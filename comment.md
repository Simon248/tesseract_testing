

## 1 What I'm looking for (simple architecture concern):
_It's not a full featured request, just basics need._
* **a serveur for planing that i can use in combination with behaviortree**
      - that could be contenerised 
      - that can be initialised by service/action call (loading urdf/srdf)
      - that provide service/action for changing the scene an robot model (tool changing, add octomap from pcl)
      - that provide action/service for feeding the program (waypoints/motion type) and another action/service for actual planning once program is feeded.
      - that will answer to plannification 
           -- with a ros trajectory_msg that can be send to a robot driver or displayed
           -- or with an error and data about the error (could be partial plannifcation to know where is the failure)
      -- with the ability to plan with constant speed or variable programmed speed (evolving along the path)
     ? instruction profiles should be fixed in the server ? would be over flexible to modify it on the go with a request.
 * Underconstrained planning
 * Underconstrained planning with 9 axis (graph based solution like descartes should explode cause too much IK solutions.)
 * Planning with tolerances (optimising a plan with orientation toleranced waypoint -- cf welding with process tolerance regarding orientation)

## 2 small remarks on sword testing
#### source ros2_wp/install/setup.bash lead to error in finding urdf mesh
```
[SWORD] No active license found: Cannot find license file. (-1,359:2 "No such file or directory")
[SWORD] DEMO mode active. Application will exit in 15 minutes.
<Exception> URDF: Error parsing file '[...]/src/description/cogniman_scene_description/urdf/robot.urdf'!
 URDF: Error parsing 'link' element for robot 'cogniman_robot'!
  Link: Error parsing 'visual' element for link 'screw_gripper'!
   Visual: Error parsing 'geometry' element!
    Geometry: Failed parsing geometry type 'mesh'!
     Mesh: Error importing meshes from filename: 'package://screw_gripper_description/mesh/visual/screw_gripper.stl'!
: [...]/urdf/robot.urdf
```
**need to export ROS_PACKAGE_PATH**

## 3 QUESTIONS/REMARKS ON SWORD:
**Usefull for testing profile** with a planner pipe.
**SRDF :** is it possible to import or export SRDF ?  
is it possible to **import program serialised from tesseract?**

## 4 TRAINING: reflexion about content

* Overview of tesseract (why? dedicated use case ?)

#### Tesseract architecture and Language
* Tesseract language structure
     - waypoints /  composite_instruction / Move_instruction / any_poly...


#### Tesseract Planning

* architecture and semantics: motion pipelinne, task, taskflow, profile, executor...

* common profile with common task.

* descripotion of the **different motion planner/Planning pipeline**

* **remote TCP** planning (part in hand and fixed tool)

* **Time parametrisation option**, how to make Constant speed traj & Variable but defined speed on traj. _i dont understand time parametrisation... if we change time parametrisation to have constant speed after planning (modify time, velocities, accel)  how can we ensure that the plan is still feasible? and if we check for feasibility (with respect to axis limits) and it's not feasible, then it should result in a failure. But maybe other plan could have success in time paramtrisation...

* **IK**: differnce between kdl LMA & KDL NR, why there is no ikfast ? 

#### SRDF
* Make SRDF (no snap for tesseract setup, just a review of pure srdf syntax)

#### other:
* Planning server:
     - is it a viable way of using tesseract with ROS? (client not updated in the repo)
     - Get feedback on planning ? (if it failed, why ?) graph provide info for developers but not user friendly.

* Modify scene graph (tool change, modify scene-> pop a part, pop a pcl (as voxel grid))

* OPW IK overview
* Planning for 9 axis (gantry + 6 axis)

* study case on Cogniman Cell (with remote TCP planning)
* Study case on 9 axis cell (gantry + 6 axis)

* how to analyse the tesseract dotgraph

### Use case studies
* **use case1:** A GP7 picking a part and  making a contact operation with fixed tool.

* **use case2:** Welding with a 9 axis robot (gantry + 6 axis). underconstrained planning. (Using sparce descartes then trajopt?)  
 explore capabilties of orientation optimisation?


## 5 OBSERVATION WIP

 ### PROFILE IN MOVE INSTRUCTION

 ![alt text](pictures/move_PROFILE.png)

### EXECUTOR and TASKFLOW
i'm not sure to get the diffrence between the taskflow and the executor.

![alt text](pictures/planning_request.png)

here are the task _(request.name in the server)_:
![alt text](pictures/task_list.png)
those task are different from motion_pipeline referenced in SWORD documentation...

![alt text](pictures/SWORD_planning_pipelines.png)

**so what is an executor?**
executor are listed through ```tesseract_planning::TaskComposerServer ->getAvailableExecutors```
only one available by default in the planning server: TaskflowExecutor
Why could I want more than one executor? 

### test 1
* my test programme:
```
Program: Composite Instruction, Description: Tesseract Composite Instruction
Program: { 
Program:     Composite Instruction, Description: from_start
Program:   {
Program:     Move Instruction, Move Type: 1, State WP: Pos=0 0 0 0 0 0, Description: Start
Program:     Move Instruction, Move Type: 1, Cart WP: xyz=0.4, 0, 1, Description: from_start_plan
Program:   }
Program:   Composite Instruction, Description: Raster # 1
Program:   {
Program:     Move Instruction, Move Type: 0, Cart WP: xyz=0.89, -0.01, 1 , Description: Tesseract Move Instruction
Program:     Move Instruction, Move Type: 0, Cart WP: xyz=0.891, -0.01, 1 , Description: Tesseract Move Instruction
Program:     Move Instruction, Move Type: 0, Cart WP: xyz=0.892, -0.01, 1 , Description: Tesseract Move Instruction
Program:   }
Program:   Composite Instruction, Description: to_end
Program:   {
Program:     Move Instruction, Move Type: 1, State WP: Pos=0 0 0 0 0 0 , Description: to_end_plan
Program:   }
Program: }
```

* using pipeline:   
```
goal_msg.request.name = "TrajOptPipeline";
```
problem: no time in tesseract trajectory

* using pipeline:   
```
goal_msg.request.name = "RasterFtPipeline";
```
```
I've got traj timming, but only integer in tesseract_common::JointTrajectory.states.time
[WARN] approximate merit function got worse (-5.058e-01). (convexification is probably wrong to zeroth order)
```

and
```
Error:   Raster subgraph failed
```
<img src="pictures/graphviz1.svg" alt="Description of the SVG">

i understand that task 'from_start' and 'to_end' are paralised, but done after Raster.  
indeed not done cause raster failed.  
I dont understand the second graph, what is SimpleMotionPlannerTask.  
SimpleMotionPlannerTask reference the raster graph which failed cause 
```
Failed to find valid solution:  OPT_PENALTY_ITERATION_LIMIT
```
i understand that trajopt reach iteration limit without solving the problem.


* ??? in the pipelines name, when you've got XXXpipeline, like in RasterFTpipelines it means that it handles multiple composites instructions (like freespace - raster - freespace). And when there is no *pielines the it use a single planner.

* ??? what is CT and FT (RasterXXpipelines for instance)

* 

* difference btw planInstruction and MoveInstruction


#### Profiles
* what is a profile ?
* why I can specifie profile in CompositeInstruction as well as in MoveInstruction
<img src="pictures/profile1.png" alt="Description of the SVG">

* a compositeInstruction can contain a compositeInstruction.
* if ManipInfo is pecified in level0 it doesn't has to be redefined in lower levels of CI.
