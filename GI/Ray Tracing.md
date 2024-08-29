# Ray Tracing



## 0. Intro

1. **Raycasting**

   Casting rays from the viewpoint (camera) into the scene. Each ray corrsponds to a pixel in the final image. The goal is to **determine which objects in the scene a ray will intersect with**.

2. **Light Simulation**

   Calculates the color of that point by simulating <font color=red>how light behaves when it hits the object</font>. This includes calculating **direct illumination** from light sources and **indirect illumination** from light bouncing off other objects.

   > - bouncing
   >   - 反射

3. **Material Properties**

   - Color, texture, reflectivity, transparency

4. **Recursion**

5. **Shadows**









## 1. Raycasting

Determine **visibility** along a line of sight from a specific point.

Basic idea: **cast** (or project) rays from a viewpoint and determine what objects those rays within the scene.



### 1.1 Outline

#### 1.1.1 **Origin and Direction**

A ray is defined by an **origin point** (the observer's location) and a **direction vector** (pointing towards where the observer is looking).



#### 1.1.2 Intersection Tests:

The algorithm checks **if and where the ray intersects objects** in the scene.  

This usually involves mathematical formulas specific to each type of object (e.g., spheres, planes, boxes).

For each object, the algorithm calculates whether the ray intersects it and at what distance from the origin.



#### 1.1.3 Visibility Determination

If a ray intersects an object, that object is considered "visible" from the ray's origin point. The **nearest** intersected object along the ray's path is typically what the observer would "see", blocking any further objects along the same line of sight.



#### 1.1.4 Rendering or Analysis

In graphics applications, the **color** and **texture** of the **visible object at the intersection point** can be used to render the scene from the observer's perspective. In other applications, the fact of intersection itself might be used for analysis, such as **obstacle detection** in robotics.



Common algorithms involve solving equations for **ray-sphere intersection**, **ray-plane intersection**, and using **spatial data structures** like <font color=red>**bounding volume hierachies (BVH)**</font> or grids to efficiently locate potential intersections.











### 1.2 Specific Algorithms and Techniques

In the context of ray casting within global illumination and computer graphics, several specific algorithms and techniques are used to optimize performance and achieve realistic lighting effects.



#### 1.2.1 Basic Ray Casting Algorithm

- **Discription**
  - The most fundamental form of ray casting involves shooting rays from the eye (camera) through each pixel on the screen and finding the first object hit by each ray. This basic approach determines the color of the pixel based on the material of the object and the lighting conditions.















































