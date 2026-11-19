Apple Toss — MuJoCo Humanoid Simulation
A 43-joint humanoid robot picks an apple off a pedestal, tosses it into the air and catches it with the other hand, then presents it. Everything you see is simulated: contacts, friction, gravity, air drag and joint dynamics. MuJoCo runs as WebAssembly in the browser, Three.js draws it.

Open the app and the robot starts working through the task on its own.

The task
Observe → Reach → Grasp → Lift → Wind-up → Toss → Flight → Catch → Present
                                                    ↘ Missed (apple on the floor)
Observe — the robot stands and looks at the apple on the pedestal.
Reach — the right arm swings in over the pedestal, torso leans, pelvis dips.
Grasp — fingers close around the apple; a weld constraint freezes it in the hand.
Lift — the palm rotates up and the apple comes to chest height.
Wind-up — the hand dips, the knees bend, the left hand opens and moves to the catch spot.
Toss — an explosive upward stroke releases the apple on a planned ballistic arc.
Flight — the apple flies freely; the head tracks it and the landing point is re-predicted every ~30 ms from its live state.
Catch — the left hand moves under the predicted landing point and cushions the impact.
Present — the left arm lifts the apple into view. If the apple falls below 45 cm without being caught, the cycle is marked Missed instead.
The cycle repeats automatically (toggle it off in the panel).

Running it
bun install        # or: npm install
bun run dev        # or: npm run dev  → open the URL it prints
Needs a browser with WebAssembly and WebGL (any current Chrome, Firefox, Edge, Safari).

Other commands:

Command	What it does
bun run dev	Dev server with the live simulation
bun run build / bun run preview	Production build and local preview
bun run test	Headless physics tests (no browser needed)
bun run lint / bun run format	ESLint / Prettier
Controls
Playback — Play/Pause, Restart, speed 0.1× / 0.25× / 0.5× / 1×, "Repeat automatically".

Toss

Toss height (25–70 cm): apex above the release point.
Hand spacing (20–40 cm): horizontal distance between release and catch hand.
Changes take effect on the next toss, and the robot re-aims itself every throw.

View — Orbit, Front, Side, and Robot (a camera mounted on the head's gaze site). Drag to orbit, scroll to zoom. "Trail & predicted path" toggles the apple's flight trail, the dashed prediction arc and the landing ring.

Telemetry — sim time, where the apple is (right hand / left hand / airborne / floor / pedestal), height, speed, release speed, peak height, flight time, catch offset in cm, catches so far, and the current aim correction.

Download MJCF — saves the exact model as apple_toss_humanoid.xml, which you can open in MuJoCo's own viewer:

python -m mujoco.viewer --mjcf apple_toss_humanoid.xml
Physics model at a glance
Solver	MuJoCo 4 (WASM), 500 Hz fixed timestep (timestep="0.002"), implicitfast integrator
Contacts	Elliptic friction cone, impratio=5, soft constraint solref="0.006 0.9"
Air	density="1.2", viscosity="0.000018" — the apple feels drag
Robot	43 actuated joints: pelvis lift, 3 per leg, 2 waist, 2 neck, 6 per arm, 10 per hand
Hands	4 fingers + thumb per side, two phalanges each, with joint limits and damping
Apple	Free joint, sphere r = 4.2 cm, mass 170 g, stem and leaf are massless decoration
Pedestal	Cylinder column + top plate at (0.34, −0.30), top surface at 0.95 m
Gravity	All robot links are gravity-compensated, like a real torque-controlled humanoid
How the joints are driven
Every joint is a MuJoCo position actuator with its own kp, kv and force limit. Each step the controller writes

ctrl = q_des + (kv / kp) · q_des_vel
so the servo sees an implicit PD term plus velocity feed-forward — that's what lets the toss stroke track a fast reference without lagging badly.

How the arms find their targets
A 6×6 damped least-squares inverse kinematics solver runs each step on a scratch copy of the model: the Jacobian comes from mj_jacSite on the palm's grip site, orientation error comes from the hand's local axis, and a weak rest-pose term keeps the elbow from drifting. Joints are clamped to their limits every iteration.

Legs use analytic two-link IK, so both feet stay planted while the pelvis rises and dips. The head follows the apple with a critically damped pursuit filter.

How the grasp works
Grabbing activates a weld equality constraint between hand and apple. The relative pose is written at the moment of contact, so the apple freezes exactly where it touched the palm instead of snapping to a fixed offset. Releasing just deactivates the weld.

How the toss is aimed
tossVelocity() solves for the release velocity that puts the apple's apex apex metres above the hand and lands it on the catch point. Then two feedback layers clean it up:

Aim learning — after each release the measured apple velocity is compared with the planned one, and the difference (×0.7) is added to a persistent correction vector. Servo lag and contact kick get compensated automatically; it converges to ~2 cm catch error within a few cycles.
Adaptive catch — during flight the landing point is re-predicted from the apple's actual position and velocity every ~30 ms, and the left hand replans to it. A wobble or an off-centre release is corrected mid-air.
Where the code lives
src/sim/
  humanoidXml.ts   MJCF builder: humanoid, hands, pedestal, apple, actuators
  math.ts          vector math, quintic/minimum-jerk easing, IK solver helper,
                   ballisticAtHeight(), tossVelocity()
  controller.ts    TossController: phase machine, IK, grasp welds, aim learning
  renderer.ts      Three.js view: meshes per geom, lights, cameras, trail, prediction arc
  engine.ts        SimEngine: loads the WASM, fixed-step loop, pause/speed/camera/telemetry
  toss.test.ts     headless pick → toss → catch test
src/components/sim/
  SimulationView.tsx  the UI: panel, sliders, telemetry, phase timeline
  phases.ts           phase names and hints
src/sim/ is plain TypeScript with no React in it, so the same controller and model run unchanged in Node tests and in the browser. The engine (WebGL + WASM) is loaded with a dynamic import() inside an effect so server-side rendering never touches it.

Tests
bun run test runs Vitest. src/sim/toss.test.ts loads the real model headless, drives the controller and MuJoCo for a full cycle, and checks that the right hand holds the apple, the apple actually flies (peak above 1.4 m), the left hand ends up holding it, and the cycle reports one catch. There's also a separate check on the toss-planning math itself.

Notes and limits
The simulation is real-time: at 1× it steps as fast as your machine allows and the panel shows the achieved real-time factor. Slower devices drop to a lower factor rather than skipping ahead.
A missed catch is a legitimate outcome, not a bug — push the toss height to the top of the slider with the widest hand spacing and you can make it happen.
The model is a hand-built MJCF (capsules, ellipsoids, boxes), not a scanned asset, so proportions are approximate on purpose.
node_modules, build output and caches are not part of the source archive; reinstall with bun install.
