Environment Builder
===================

Environment compilation is the step that turns the three independent components —
scene, embodiment, and task — into a runnable Isaac Lab environment.
``ArenaEnvBuilder`` does this by collecting the partial configuration each
component contributes and merging them into a single
``ManagerBasedRLEnvCfg``.

.. figure:: ../../../images/arena_env_builder.png
   :width: 100%
   :alt: ArenaEnvBuilder merges Scene, Embodiment, and Task into a ManagerBasedRLEnv
   :align: center

   ``ArenaEnvBuilder`` merges the Scene, Embodiment, and Task into a runnable ``ManagerBasedRLEnv``.

.. code-block:: python

   environment = IsaacLabArenaEnvironment(
       name="manipulation_task",
       embodiment=embodiment,
       scene=scene,
       task=task,
   )

   env_builder = ArenaEnvBuilder(environment, ArenaEnvBuilderCfg())
   env = env_builder.make_registered()

How it works
------------

Each component (Scene, Embodiment, Task) exposes a set of ``get_*_cfg()`` methods that return its
contribution to each Isaac Lab manager. The typical contributions of each component
to each manager are tabulated below:

+-------------+------------------------------------------------------------------------+
| Isaac Lab   | Isaac Lab - Arena Component                                            |
+ Manager     +-----------------------+-------------------------+----------------------+
|             | Scene                 | Embodiment              | Task                 |
+=============+=======================+=========================+======================+
| Scene       | assets, lights        | robot, sensors          | task-specific assets |
+-------------+-----------------------+-------------------------+----------------------+
| Observations|                       | proprioception, cameras | goal observations    |
+-------------+-----------------------+-------------------------+----------------------+
| Actions     |                       | control interface       |                      |
+-------------+-----------------------+-------------------------+----------------------+
| Events      | object placement      | robot reset             | task reset           |
| (resets)    |                       |                         |                      |
+-------------+-----------------------+-------------------------+----------------------+
| Terminations|                       |                         | success, failure     |
+-------------+-----------------------+-------------------------+----------------------+
| Rewards     |                       |                         | dense rewards (RL)   |
+-------------+-----------------------+-------------------------+----------------------+
| Recorder    |                       |                         | metrics-required data|
+-------------+-----------------------+-------------------------+----------------------+


``ArenaEnvBuilder.build_env_cfg()`` first assembles the partial manager contributions
from each component into a set of complete managers. Then it merges these complete managers
into a single ``ManagerBasedRLEnvCfg``.

By default, the builder also solves spatial relations for placed objects and
supported robot embodiments. Set
``ArenaEnvBuilderCfg(solve_relations=False)`` in Python to disable this step.
Arena command-line runners expose the equivalent ``--no_solve_relations``
option.

Seeds during compilation
------------------------

Environment compilation and rollout use more than one random stream, so locking
a single global seed is not enough when you need layouts, object-set picks, or
variation values to be reproducible independently.

What it is
~~~~~~~~~~

``ArenaEnvBuilder`` exposes two seeds. They are independent — locking one does
not fix the other.

.. list-table::
   :header-rows: 1
   :widths: 18 28 12 42

   * - Control
     - CLI / config
     - Default
     - Locking it reproduces
   * - Environment seed
     - ``--seed`` / ``ArenaEnvBuilderCfg.seed``
     - ``42``
     - Build-time and run-time :doc:`variation <../variations/variations>`
       values, and the simulation RNG (for example reset noise).
   * - Placement seed
     - ``--placement_seed`` / ``ArenaEnvBuilderCfg.placement_seed``
     - ``None`` (unlocked)
     - Relation-solver layouts and random
       :doc:`RigidObjectSet <../scene/concept_rigid_object_set>` member
       assignment. With ``None``, placement stays non-reproducible across runs.

There is no separate variation seed: each variation draws from its own
generator, seeded from ``--seed`` and the variation's
``<asset_name>.<variation_name>`` key. In-order
:ref:`asset selection <asset-selection-variation>` does not use the seed. It
takes the name at the rebuild index, which the Experiment Runner passes to
``ArenaEnvBuilder`` for each rebuild. Run-time variations hand out their
values in episode-start order, so the k-th episode to start gets the same
values for any ``num_envs``. Which environment runs it still depends on
``num_envs`` and on when environments reset. See
:doc:`../variations/variations` and
:doc:`../object_placement/pooled_placement`.

How to set it
~~~~~~~~~~~~~

Pass the seed you want to lock on the runner CLI (or set the matching field on
``ArenaEnvBuilderCfg`` / ``placer_params``):

.. code-block:: bash

   python isaaclab_arena/evaluation/policy_runner.py \
     --policy_type zero_action \
     --seed 42 \
     --placement_seed 7 \
     --num_steps 100 \
     pick_and_place_maple_table

- Set ``--seed`` to fix variation values and simulation draws.
- Set ``--placement_seed`` to fix layouts and random object-set picks.
- Omit ``--placement_seed`` when placement should vary across runs.

The compiled config is then registered with the gym registry under the
environment's name, and ``gym.make()`` returns the gym environment.

Explicit episode resets
-----------------------

By default, ``IsaacLabArenaManagerBasedRLEnvCfg.auto_reset`` is ``True``: an
ended episode is replaced automatically during ``step()``. Set it to ``False``
when the caller must decide when each native batch slot starts its next episode.
Configure this before creating the environment:

.. code-block:: python

   cfg, kwargs = env_builder.build_env_cfg()
   cfg.auto_reset = False
   registered_env = env_builder.make_registered(cfg, kwargs)
   native_env = registered_env.unwrapped

   observations, info = native_env.reset(seed=42)
   # After stepping with task-compatible actions, reset only the completed slots.
   observations, info = native_env.reset(env_ids=[0], seed=43)
   native_env.close()

``make_registered()`` returns a Gym wrapper whose ``reset()`` does not accept
``env_ids``. Use the underlying native environment consistently for reset and
step when implementing this lifecycle; do not bypass the wrapper for reset and
then call its reset-enforcing ``step()``.

With automatic reset disabled, each completion is recorded once and no replacement
episode starts until an explicit reset. The environment advertises disabled
autoreset metadata. ``configure_episode_limit()`` requires automatic replacements
and cannot be combined with this mode.

.. note::

   Disabling replacement episodes does not freeze completed bodies in the shared
   physics world. A vector adapter that promises persistent terminal observations
   must retain the first terminal observation and outcome itself while peers step.
   A selected reset applies its seed to that call; it does not create independent
   random streams for all subsequent simulation events.

Mimic mode
----------

Passing ``--mimic`` at the command line compiles a
``ManagerBasedRLMimicEnv`` instead of a standard ``ManagerBasedRLEnv``.
The mimic environment is used for demonstration generation and includes
subtask configurations from the task. Metrics and recorders are excluded
in mimic mode.

.. code-block:: bash

   python submodules/IsaacLab/scripts/imitation_learning/isaaclab_mimic/generate_dataset.py \
     --external_callback isaaclab_arena.environments.isaaclab_interop.environment_registration_callback \
     --mimic ...

Next Steps
----------

Continue to :doc:`../object_placement/relations` to learn how anchors and spatial
relations describe a placement layout.
