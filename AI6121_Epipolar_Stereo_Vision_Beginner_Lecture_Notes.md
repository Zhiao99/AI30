# AI6121 Computer Vision — Epipolar Geometry and Stereo Vision

**Beginner-friendly illustrated lecture notes**  
**Based on:** *AI6121 Computer Vision, Lecture 11: Epipolar & Stereo Vision* (58-slide course handout, 2026)  
**Audience:** Students with no previous background in computer vision  
**Estimated teaching time:** 2–3 hours plus exercises

> **How to read these notes.** First understand the picture and the everyday analogy. Then read the definition, work through the equations, and try the example. Images labeled **Course slide** are reproduced from the supplied lecture PDF. Sections labeled **Extra explanation** provide additional teaching context beyond the slides. Mathematical notation is standardized throughout: unprimed symbols denote the **left/first camera** and primed symbols the **right/second camera**.

## Learning objectives

By the end, you should be able to explain why one photograph cannot normally reveal absolute depth; define a baseline, epipole, epipolar plane, epipolar line, parallax, and disparity; derive depth from disparity for parallel cameras; describe how image correspondences are found; use the fundamental and essential matrices to express epipolar constraints; outline the normalized eight-point algorithm and RANSAC; and explain how matched points and known camera matrices produce a 3D point.

---

# 1. Introduction: Why do we need two cameras?

## 1.1 A photograph loses depth

Imagine taking a picture of a toy car. In the photograph, a **small car close to the camera** might occupy exactly the same number of pixels as a **large car far away**. The picture alone does not uniquely tell us the true size and distance.

![Perspective projection and the depth ambiguity (course slide 3)](epipolar_lecture_images/slide_03.png)

*Figure 1 — Course slide 3. Different 3D points on the same viewing ray can project to the same image location.*

A simple camera model is the **pinhole camera**. A 3D point in camera coordinates is \(P=(X,Y,Z)\), and its 2D projection is \((x,y)\):

$$x=f\frac{X}{Z},\qquad y=f\frac{Y}{Z}.$$

**Definitions**

- \(X,Y\): horizontal and vertical location of a point relative to the camera.
- \(Z\): **depth**, the distance along the camera's forward axis (not necessarily the straight-line distance to the camera).
- \(f\): **focal length**, the distance from the pinhole to the ideal image plane; use consistent units.
- \(x,y\): coordinates of the projected point on the image plane.

**Why depth is ambiguous:** replace \((X,Y,Z)\) with \((2X,2Y,2Z)\). Both ratios \(X/Z\) and \(Y/Z\) stay unchanged, so the pixel position stays unchanged. A single pixel therefore corresponds to an entire **ray** in 3D, not just one 3D location.

> **Everyday analogy:** Looking through one eye, you cannot easily tell whether a distant object is small or simply far away. Opening your second eye gives you another viewpoint.

### Extra explanation: What is a camera matrix?

A camera is commonly described by a **projection matrix** \(P\):

$$\tilde{\mathbf x}\sim P\tilde{\mathbf X},\qquad P=K[R\mid t].$$

Here \(\tilde{\mathbf X}=(X,Y,Z,1)^T\) is a 3D point in **homogeneous coordinates**, \(\tilde{\mathbf x}=(u,v,1)^T\) is its image position, \(K\) is the **intrinsic matrix** (focal lengths and principal point), and \(R,t\) describe the camera's rotation and translation relative to a chosen world coordinate frame. The symbol \(\sim\) means *equal up to a nonzero scale factor*. Homogeneous coordinates allow perspective projection to be represented using matrix multiplication.

For a simple camera with zero skew:

$$K=\begin{bmatrix}f_x&0&c_x\\0&f_y&c_y\\0&0&1\end{bmatrix}.$$

The principal point \((c_x,c_y)\) is approximately where the optical axis meets the image. **Pixels** \((u,v)\) and **normalized camera coordinates** \((x_n,y_n)\) are related by \(\tilde{\mathbf x}_n=K^{-1}\tilde{\mathbf x}\).

## 1.2 How two cameras help

If two cameras view the same scene from different positions, a physical point appears at two image locations. Each location gives a viewing ray. When the cameras' relative positions are known, we can intersect (or approximately intersect) those rays to estimate the point's 3D location. This is called **triangulation**.

The course divides stereo geometry into three questions:

| Question | Name | What are we trying to find? |
|---|---|---|
| Where should the same point appear in the other image? | **Correspondence geometry** | A matching point or its allowed line |
| How are the two cameras positioned relative to each other? | **Camera geometry / motion** | Camera rotation and translation |
| Where is the real-world point? | **Scene geometry / structure** | The point's 3D coordinates |

**Key idea:** Two cameras provide complementary measurements. However, simply having two pictures is not enough: we also need correct correspondences and sufficient camera geometry.

---

# 2. Epipolar geometry: The geometry connecting two views

## 2.1 Five essential definitions

![Epipolar geometry diagram (course slide 7)](epipolar_lecture_images/slide_07.png)

*Figure 2 — Course slide 7. The two camera centers, baseline, 3D point, image points, epipoles, and epipolar lines.*

1. **Camera center** \(C\) or \(C'\): the viewpoint from which the camera's rays originate.
2. **Baseline**: the line connecting the two camera centers. Its physical length is often written \(B\) (the original slides also use \(T\)).
3. **Epipolar plane**: the plane passing through both camera centers and a particular 3D scene point \(X\).
4. **Epipolar line** \(\ell\) or \(\ell'\): the line where an epipolar plane intersects an image plane.
5. **Epipole** \(e\) or \(e'\): the image of the *other* camera's center; equivalently, the intersection of the baseline with the image plane in projective geometry.

**Important distinction:** An **epipole is a point**, an **epipolar line is a line**, and an **epipolar plane is a plane in 3D**. All epipolar lines in a given image pass through that image's epipole (possibly a point at infinity).

## 2.2 Why the matching point must lie on a line

Suppose you choose a pixel \(\mathbf x\) in the left image. The corresponding 3D point must lie somewhere on the ray from the left camera center through that pixel. This ray and the two camera centers determine an epipolar plane. The plane cuts the right image along one epipolar line \(\ell'\). Therefore, the matching right-image point \(\mathbf x'\) **must lie on \(\ell'\)**.

![Epipolar constraint illustration (course slide 8)](epipolar_lecture_images/slide_08.png)

*Figure 3 — Course slide 8. A selected left-image point constrains the right-image match to a single epipolar line.*

**Why this matters:** Without geometry, you might search every pixel in a 1920 × 1080 image—over two million candidates. With the epipolar constraint, you only search along a line. This greatly reduces the search space, although it does **not** tell you which point on that line is correct.

**Check your understanding:** Does the epipolar constraint guarantee a unique match? **No.** Repeated textures, occlusions, and similar-looking pixels can still cause ambiguity.

---

# 3. Stereo vision with parallel cameras

## 3.1 The simplest stereo setup

Imagine two cameras placed side by side, looking straight ahead with the same focal length. Their optical axes are parallel and their image rows are aligned. This idealized arrangement is called **rectified stereo** when achieved by transforming the images; the course first introduces it through physically parallel cameras.

In this setup, corresponding points ideally have the same vertical image coordinate, and epipolar lines are horizontal. The epipoles are at infinity in the horizontal direction.

![Parallax (course slide 10)](epipolar_lecture_images/slide_10.png)

*Figure 4 — Course slide 10. A foreground object appears to shift relative to the background when viewed from two positions.*

### Parallax versus disparity

- **Parallax** is the apparent change in position of an object when the viewpoint changes; it is a general visual/geometric phenomenon.
- **Disparity** is the measured difference between the locations of a corresponding point in two stereo images, usually expressed in pixels after rectification.

For rectified horizontal stereo:

$$d=u_L-u_R.$$

The sign depends on which camera is labeled left and on the image-coordinate convention. In the standard arrangement used below, a point in front of the cameras has positive disparity.

## 3.2 Deriving depth from disparity

![Stereo triangulation geometry (course slide 11)](epipolar_lecture_images/slide_11.png)

*Figure 5 — Course slide 11. Similar triangles lead to the stereo depth equation.*

Let the left camera center be \((0,0,0)\) and the right camera center be \((B,0,0)\). A scene point is \((X,Y,Z)\). With focal length \(f\) and principal-point-centered image coordinates:

$$x_L=f\frac{X}{Z},\qquad x_R=f\frac{X-B}{Z}.$$

Subtract the second equation from the first:

$$d=x_L-x_R=\frac{fX}{Z}-\frac{f(X-B)}{Z}=\frac{fB}{Z}.$$

Rearrange:

$$\boxed{Z=\frac{fB}{d}}$$

**Interpretation:** Larger disparity means a point is closer. Smaller disparity means a point is farther away. The formula assumes parallel/rectified cameras, equal effective focal lengths, aligned principal-point conventions, and a known baseline. Use \(f\) in pixels if \(d\) is in pixels; \(B\) and \(Z\) then share the same physical unit.

### Worked example 1: Find an object's depth

Suppose \(f=800\) pixels, \(B=0.12\) m, and \(d=40\) pixels. Then:

$$Z=\frac{800\times0.12}{40}=\boxed{2.4\ \text{m}}.$$

If another object has \(d=20\) pixels, its depth is \(4.8\) m. **Halving disparity doubles depth**, with the same camera setup.

## 3.3 Why distant objects are harder to measure accurately

![Stereo depth uncertainty derivation (course slide 12)](epipolar_lecture_images/slide_12.png)

*Figure 6 — Course slide 12. Differentiating the depth equation shows how disparity errors propagate into depth errors.*

Starting from \(Z=fB/d\), differentiate with respect to disparity:

$$\frac{\partial Z}{\partial d}=-\frac{fB}{d^2}=-\frac{Z^2}{fB}.$$

For a small disparity error \(\delta d\), first-order error propagation gives:

$$\boxed{|\delta Z|\approx\frac{Z^2}{fB}|\delta d|}.$$

The square of \(Z\) is crucial: **depth uncertainty increases approximately quadratically with distance** for a fixed disparity uncertainty.

### Worked example 2: One-pixel error

With \(f=800\) pixels, \(B=0.12\) m, and \(|\delta d|=1\) pixel:

| Actual depth | Approximate depth error |
|---|---:|
| 2 m | 0.042 m |
| 5 m | 0.260 m |
| 10 m | 1.042 m |

This is a **local approximation**, not an exact finite-error bound. It shows why measuring distant objects with stereo becomes difficult.

### The baseline trade-off

A **larger baseline** increases disparity and usually improves geometric depth precision. But it also creates larger appearance differences, more occlusions, and harder point matching. A **smaller baseline** makes images easier to match but gives weaker depth information. The original lecture emphasizes that practical stereo systems must balance these two effects.

---

# 4. Finding corresponding points

The depth formula is easy to apply **after** disparity is known. The difficult question is: *Which pixel in the right image represents the same real-world point as a chosen pixel in the left image?*

## 4.1 Appearance-based matching

**Definition:** Compare the appearance of a small patch around the left pixel against candidate patches in the right image. Choose the most similar one.

A simple measure is **sum of squared differences (SSD)**:

$$\operatorname{SSD}(x,y)=\sum_{i=-N}^{N}\sum_{j=-N}^{N}\left[I_R(x+i,y+j)-g(i,j)\right]^2,$$

where \(g\) is the reference patch from the left image, and \(I_R\) is the right image. Smaller SSD means a better match under the model's assumptions.

![Stereo patch matching (course slide 15)](epipolar_lecture_images/slide_15.png)

*Figure 7 — Course slide 15. Candidate matches are tested along a scanline; the lowest SSD score suggests the best disparity.*

**Toy example:** If three candidate patches have SSD scores 150, 12, and 90, the second patch is preferred. The matching disparity is the horizontal shift from the reference pixel to that candidate.

**Limitations:** SSD can fail when illumination changes, the viewpoint changes substantially, the patch contains a repeated pattern, or the point is hidden in the other view (**occlusion**).

## 4.2 Disparity maps

A **disparity map** stores the estimated disparity for each pixel where a valid match can be found. It is a 2D image representing horizontal shifts, not depth directly. For a standard positive-disparity setup, larger disparity generally indicates a nearer surface.

![Stereo pair and disparity map (course slide 16)](epipolar_lecture_images/slide_16.png)

*Figure 8 — Course slide 16. A stereo image pair is used to construct a disparity map. Display brightness depends on the visualization scale.*

### Extra explanation: From disparity map to depth map

For each pixel with valid positive disparity, compute \(Z(u,v)=fB/d(u,v)\). Invalid or near-zero disparity must be handled carefully because division by zero is undefined and tiny disparity produces very unstable depth estimates.

## 4.3 Feature-based matching

Instead of comparing raw pixel intensities, we can identify distinctive **features** such as corners, edges, and textured regions, then compare descriptors of those features between images.

The lecture highlights **SIFT (Scale-Invariant Feature Transform)**, which detects characteristic keypoints and describes their local image patterns using a **128-dimensional feature vector**. SIFT is designed to be relatively robust to changes in scale and in-plane rotation, and often tolerates moderate changes in viewpoint and illumination.

![SIFT descriptor example (course slide 19)](epipolar_lecture_images/slide_19.png)

*Figure 9 — Course slide 19. Local image-gradient information is summarized into a SIFT descriptor.*

| Method | Main idea | Good for | Weakness |
|---|---|---|---|
| Patch matching (SSD) | Compare raw pixel values | Similar viewpoints, simple rectified stereo | Lighting/viewpoint sensitivity |
| Feature matching (e.g. SIFT) | Compare distinctive descriptors | Sparse matches across larger viewpoint changes | May produce false matches; not dense by itself |

**Extra explanation:** A *keypoint* is an interesting image location. A *descriptor* is a numerical summary of the neighborhood around that location. A good descriptor makes the same physical feature look similar across images even when the images are not identical.

---

# 5. The fundamental matrix: Expressing epipolar geometry mathematically

## 5.1 From a point to a line

A line in a 2D image can be written as:

$$au+bv+c=0.$$

Using homogeneous coordinates, let \(\mathbf\ell=(a,b,c)^T\) and \(\mathbf x=(u,v,1)^T\). Then a point lies on the line exactly when:

$$\mathbf x^T\mathbf\ell=0.$$

**Definition — Fundamental matrix \(F\):** A \(3\times3\) matrix encoding the epipolar geometry between two camera images. For a point \(\mathbf x\) in the first image, its corresponding epipolar line in the second image is:

$$\boxed{\mathbf\ell'=F\mathbf x}.$$

The matching point \(\mathbf x'\) must lie on that line:

$$\boxed{\mathbf x'^T F\mathbf x=0}.$$

This is the **epipolar constraint**. It does **not** give the matching pixel directly; it provides a geometric condition that a valid match must satisfy.

![Fundamental matrix mapping illustration (course slide 32)](epipolar_lecture_images/slide_32.png)

*Figure 10 — Course slide 32. A point in the left image maps to a line in the right image.*

**Notation caution:** Some original slides place the primed point on the left side of the matrix product or use a transposed matrix convention. These notes consistently use \(\mathbf x'^TF\mathbf x=0\), so \(\ell'=F\mathbf x\) and \(\ell=F^T\mathbf x'\). Both conventions are valid if used consistently.

## 5.2 What does rank 2 mean?

The fundamental matrix is **singular**:

$$\operatorname{rank}(F)=2,\qquad \det(F)=0.$$

A general 3×3 matrix has nine entries. Because multiplying \(F\) by any nonzero scalar represents the same epipolar constraint, one degree of freedom is redundant. The rank-2 constraint removes another degree of freedom. Therefore, \(F\) has **7 degrees of freedom**.

The epipoles are related to the null spaces:

$$F\mathbf e=0,\qquad F^T\mathbf e'=0.$$

Here \(\mathbf e\) is the epipole in the **first** image and \(\mathbf e'\) is the epipole in the **second** image under our chosen convention. This also explains why all corresponding epipolar lines pass through the epipoles.

## 5.3 Essential matrix versus fundamental matrix

When the camera intrinsics are known, we can remove their effect and work in **normalized camera coordinates**:

$$\mathbf x_n=K^{-1}\mathbf x,\qquad \mathbf x_n'=K'^{-1}\mathbf x'.$$

The **essential matrix** \(E\) relates these normalized coordinates:

$$\mathbf x_n'^T E\mathbf x_n=0.$$

With relative camera pose \(\mathbf X_2=R\mathbf X_1+t\),

$$\boxed{E=[t]_\times R}.$$

The symbol \([t]_\times\) denotes a skew-symmetric matrix that computes the cross product:

$$[t]_\times=\begin{bmatrix}0&-t_z&t_y\\t_z&0&-t_x\\-t_y&t_x&0\end{bmatrix},\qquad [t]_\times v=t\times v.$$

![Essential matrix derivation (course slide 25)](epipolar_lecture_images/slide_25.png)

*Figure 11 — Course slide 25. Relative camera motion leads to the essential-matrix constraint.*

Substituting normalized coordinates into the essential-matrix constraint gives:

$$\mathbf x'^T K'^{-T} E K^{-1}\mathbf x=0.$$

Hence:

$$\boxed{F=K'^{-T}EK^{-1}},\qquad \boxed{E=K'^TFK}.$$

| Property | Fundamental matrix \(F\) | Essential matrix \(E\) |
|---|---|---|
| Coordinates | Image pixels (homogeneous) | Normalized camera coordinates |
| Camera intrinsics required to estimate directly? | No | Yes, to normalize pixels |
| Matrix size | 3×3 | 3×3 |
| Rank | 2 | 2 |
| Degrees of freedom | 7 | 5 |
| Additional singular-value property | One singular value is zero | Two nonzero singular values are equal |

**Extra explanation:** The essential matrix has 5 degrees of freedom because relative rotation has 3, while translation contributes only its **direction** (2). Its absolute length cannot be recovered from two monocular views alone without additional scale information.

**Memory aid:** **F** is for image **F**eatures in pixel coordinates; **E** is for calibrated camera g**E**ometry. This is a mnemonic, not a formal definition.

---

# 6. Estimating the fundamental matrix from image matches

The course develops the **eight-point algorithm**, then improves it through **normalization** and **RANSAC**.

## 6.1 Why eight point pairs?

Suppose we have corresponding image coordinates:

$$\mathbf x_i=(u_i,v_i,1)^T,\qquad \mathbf x_i'=(u_i',v_i',1)^T.$$

Expanding \(\mathbf x_i'^TF\mathbf x_i=0\) gives:

$$u_i'u_iF_{11}+u_i'v_iF_{12}+u_i'F_{13}+v_i'u_iF_{21}+v_i'v_iF_{22}+v_i'F_{23}+u_iF_{31}+v_iF_{32}+F_{33}=0.$$

Each correspondence gives **one linear equation** in the nine entries of \(F\). Stack all matches into a matrix equation:

$$A\mathbf f=0,$$

where each row of \(A\) is:

$$[u'u,\ u'v,\ u',\ v'u,\ v'v,\ v',\ u,\ v,\ 1],$$

and \(\mathbf f=(F_{11},F_{12},F_{13},F_{21},F_{22},F_{23},F_{31},F_{32},F_{33})^T\).

The solution is defined only up to scale. **At least eight suitably distributed, nondegenerate correspondences** are used for the linear eight-point method. More than eight are usually preferable when measurements contain noise.

![Eight correspondences (course slide 35)](epipolar_lecture_images/slide_35.png)

*Figure 12 — Course slide 35. Eight matched feature locations provide eight epipolar equations.*

## 6.2 What is singular value decomposition (SVD)?

**Extra explanation:** SVD factors a matrix into \(A=U\Sigma V^T\). The diagonal of \(\Sigma\) contains nonnegative **singular values**. For the homogeneous least-squares problem \(A\mathbf f\approx0\) with \(\|\mathbf f\|=1\), the best solution is the **right singular vector associated with the smallest singular value**.

This gives an initial estimate \(\widehat F\). But noise usually makes it full rank, whereas a valid fundamental matrix must have rank 2.

## 6.3 Enforcing the rank-2 constraint

Compute the SVD of the initial estimate:

$$\widehat F=U\operatorname{diag}(s_1,s_2,s_3)V^T.$$

Set the smallest singular value to zero:

$$\boxed{F_{\mathrm{rank2}}=U\operatorname{diag}(s_1,s_2,0)V^T}.$$

This finds the nearest rank-2 matrix to \(\widehat F\) in Frobenius norm. It is a matrix correction, **not** a guarantee of perfect point matches.

![Rank correction example (course slide 38)](epipolar_lecture_images/slide_38.png)

*Figure 13 — Course slide 38. Enforcing a valid fundamental-matrix structure improves consistency of epipolar lines.*

## 6.4 Why normalization improves accuracy

Raw image coordinates can be hundreds or thousands of pixels. Terms like \(uu'\) may be millions, while the constant term is 1. This creates an **ill-conditioned** linear system: numerical errors can strongly affect the estimated solution.

The **normalized eight-point algorithm** first translates and scales points separately in each image:

1. Compute the centroid of the points.
2. Shift the centroid to the origin.
3. Scale the points so their **root-mean-square distance** from the origin is \(\sqrt2\) (equivalently, their **mean squared distance** is 2).
4. Estimate a matrix \(F_n\) from the normalized points.
5. Enforce rank 2 on \(F_n\).
6. Convert back to original pixel coordinates.

If \(\mathbf q=T\mathbf x\) and \(\mathbf q'=T'\mathbf x'\), then:

$$\mathbf q'^T F_n\mathbf q=\mathbf x'^T(T'^TF_nT)\mathbf x=0,$$

so the **denormalization formula** is:

$$\boxed{F=T'^TF_nT}.$$

**Course observation:** In the slide example, normalization reduces reported mean epipolar errors from roughly 10.0 and 9.1 pixels to 1.0 and 0.9 pixels. These are results for that example, not guaranteed performance for every dataset.

## 6.5 RANSAC: Handling incorrect matches

Even good feature matchers produce some **outliers**: point pairs that do not represent the same real-world feature. Fitting \(F\) to all matches indiscriminately can give a bad result.

**RANSAC** stands for **Random Sample Consensus**. It repeatedly builds a model from a small random sample and checks how many of the remaining observations agree with that model.

**RANSAC procedure for \(F\) (as taught in the slides):**

1. Detect feature points in both images and generate tentative matches.
2. Randomly choose 8 tentative matches.
3. Estimate a candidate \(F\).
4. Measure each match's geometric consistency with the candidate \(F\).
5. Count matches whose errors are below a threshold: these are **inliers**.
6. Repeat and retain the candidate with the strongest consensus.
7. Re-estimate \(F\) using the inliers.

![RANSAC demonstration (course slide 49)](epipolar_lecture_images/slide_49.png)

*Figure 14 — Course slide 49. RANSAC tests candidate models and counts the data points consistent with each one.*

**Extra explanation: How do we measure error?** A simple measure is the perpendicular distance from the candidate matching point \(\mathbf x'\) to its epipolar line \(\ell'=F\mathbf x=(a,b,c)^T\):

$$\operatorname{distance}(\mathbf x',\ell')=\frac{|au'+bv'+c|}{\sqrt{a^2+b^2}}.$$

A commonly used alternative is the **Sampson error**, which approximately accounts for uncertainty in both images. A match with low error is geometrically plausible, but geometry alone does not prove that its visual identity is correct.

### Example: Why RANSAC helps

Suppose you have 100 tentative matches, of which 70 are correct and 30 are wrong. A single fit to all 100 may be distorted by the 30 incorrect pairs. RANSAC tries many eight-match subsets; some subsets contain enough correct pairs to yield a plausible geometry, and the 70 correct matches can then support that geometry.

---

# 7. Reconstructing 3D points from two images

## 7.1 What information do we need?

To reconstruct a 3D point, we need (a) a matched image point in each view and (b) the cameras' projection matrices, or enough equivalent camera geometry. This step is **triangulation**.

In the special parallel-camera case, the formula \(Z=fB/d\) directly gives depth. For general camera orientations, we use the two projection equations.

## 7.2 General linear triangulation

Let the two camera matrices be \(P\) and \(P'\), with rows \(p_1^T,p_2^T,p_3^T\) and \(p_1'^T,p_2'^T,p_3'^T\). For a 3D homogeneous point \(\tilde X=(X,Y,Z,1)^T\):

$$\tilde x\sim P\tilde X,\qquad \tilde x'\sim P'\tilde X.$$

If the observed pixels are \((u,v)\) and \((u',v')\), rearrange the perspective equations to obtain:

$$\begin{bmatrix}
u p_3^T-p_1^T\\v p_3^T-p_2^T\\u'p_3'^T-p_1'^T\\v'p_3'^T-p_2'^T
\end{bmatrix}\tilde X=0.$$

The four equations form a homogeneous linear system \(A_X\tilde X=0\). Solve using SVD, then divide by the last homogeneous coordinate to recover \((X,Y,Z)\).

**Equivalent course-slide formulation:** The original lecture chooses a normalization of the camera matrices that makes the last coefficient 1, then rearranges the four equations into \(W[X,Y,Z]^T=q\) and solves by the pseudoinverse:

$$\boxed{\begin{bmatrix}X\\Y\\Z\end{bmatrix}=W^+q}.$$

For a full-column-rank \(W\), \(W^+=(W^TW)^{-1}W^T\). In numerical software, use a stable least-squares or SVD solver rather than explicitly forming \((W^TW)^{-1}\).



## 7.3 Why triangulation is not always perfect

Real image points contain localization errors. Therefore, the two back-projected rays may not intersect exactly. A least-squares triangulation finds an approximate 3D point consistent with the measurements.

Typical failure cases include tiny baseline, very distant points, poor camera calibration, inaccurate feature matches, points behind a camera, and near-parallel viewing rays.

### Extra explanation: Reprojection error

After estimating a 3D point \(\widehat X\), project it back into both images using the camera matrices. Compare the predicted pixel positions with the measured pixels. The difference is called **reprojection error**. Low reprojection error is evidence of consistency, but it should be interpreted alongside camera geometry and data quality.

---

# 8. Putting everything together: A practical stereo pipeline

```text
Left image                      Right image
     |                               |
     +---------- Detect features ---+
                       |
               Match descriptors
                       |
               Tentative matches
                       |
             Estimate F with RANSAC
                       |
              Keep geometric inliers
                       |
        Calibrate / rectify when needed
                       |
         Find disparities or triangulate
                       |
              Estimate 3D structure
                       |
          Check reprojection consistency
```

**Important distinction:** A fundamental matrix can be estimated without knowing camera intrinsics, but **metric 3D reconstruction** generally requires additional calibration and scale information. The pipeline above combines the original lecture's steps with practical implementation guidance.

### Example application: A mobile robot

A robot equipped with two cameras observes a chair. It detects corresponding chair corners, rejects bad matches using RANSAC, and uses the known baseline and camera calibration to estimate their 3D positions. It can then estimate the chair's distance and plan to avoid it. A robot does not need to understand the chair's semantic identity to estimate the depth of its visible points.

---

# 9. Common misconceptions

| Misconception | Correct understanding |
|---|---|
| “One pixel tells us one 3D point.” | A pixel determines a 3D viewing ray; depth is still unknown. |
| “The epipolar line tells us the exact match.” | It limits the match to a line, but appearance or other evidence is still needed. |
| “Parallax and disparity are identical.” | Parallax is the broader apparent-shift phenomenon; disparity is a measured image-coordinate difference. |
| “Larger disparity means farther away.” | Under standard rectified stereo, larger positive disparity means **closer**. |
| “A larger baseline is always better.” | It improves geometric sensitivity but makes matching and visibility harder. |
| “F and E are interchangeable.” | F uses pixel coordinates; E uses calibrated normalized coordinates. |
| “Eight matches always guarantee a good F.” | Matches must be correct and nondegenerate; noise and outliers require care. |
| “A rank-2 F means all correspondences are correct.” | Rank 2 is a structural condition, not a guarantee of good data. |
| “Two uncalibrated images automatically give metric distances.” | Absolute scale and camera geometry need additional information. |

---

# 10. Practice questions with solutions

### Q1. Perspective ambiguity

**Question:** Why can \((X,Y,Z)=(1,2,5)\) and \((2,4,10)\) produce the same projected pixel in a pinhole camera?

**Answer:** Their \(X/Z\) and \(Y/Z\) ratios are identical. Perspective projection depends on those ratios, not the absolute 3D scale.

### Q2. Stereo depth

**Question:** A rectified stereo rig has \(f=1000\) pixels, baseline \(B=0.2\) m, and disparity \(d=50\) pixels. Find depth.

**Solution:** \(Z=fB/d=1000(0.2)/50=\boxed{4\text{ m}}\).

### Q3. Compare depths

**Question:** Two points have disparities of 80 and 20 pixels. Which is closer, assuming the same stereo rig?

**Answer:** The 80-pixel point is closer. Its depth is one quarter of the 20-pixel point's depth.

### Q4. Epipolar constraint

**Question:** If \(F\mathbf x=(2,-1,5)^T\), what equation must the corresponding point \((u',v')\) satisfy?

**Solution:** \(\ell'=(2,-1,5)^T\), so \(\boxed{2u'-v'+5=0}\).

### Q5. Fundamental matrix degrees of freedom

**Question:** Why does a fundamental matrix have 7 degrees of freedom instead of 9?

**Answer:** Its overall scale does not matter (minus 1), and it must have rank 2 (one further independent constraint): \(9-1-1=7\).

### Q6. RANSAC

**Question:** Why might estimating \(F\) using all tentative feature matches produce a poor result?

**Answer:** Some tentative matches are wrong. RANSAC identifies a subset of matches consistent with one geometric model and refits the model using those inliers.

### Q7. Conceptual challenge

**Question:** If disparity approaches zero, what happens to \(Z=fB/d\)? What does this mean physically?

**Answer:** The estimated depth grows without bound. Very distant points have tiny disparities, making their metric depths difficult to measure accurately.

### Q8. Advanced challenge

**Question:** Can we estimate \(F\) without knowing \(K\) and \(K'\)? Can we recover metric depth from \(F\) alone?

**Answer:** We can estimate \(F\) from image correspondences without camera intrinsics. But \(F\) alone does not determine absolute metric scale or uniquely provide a metric 3D reconstruction.

---

# 11. One-page formula sheet

| Topic | Formula | Meaning |
|---|---|---|
| Pinhole projection | \(x=fX/Z,\ y=fY/Z\) | Project 3D to image plane |
| Camera projection | \(\tilde x\sim K[R\mid t]\tilde X\) | General camera model |
| Disparity | \(d=u_L-u_R\) | Horizontal shift in rectified stereo |
| Depth | \(Z=fB/d\) | Depth from disparity |
| Depth error | \(|\delta Z|\approx Z^2|\delta d|/(fB)\) | First-order uncertainty |
| Epipolar line | \(\ell'=F\mathbf x\) | Line in right image from left point |
| Epipolar constraint | \(\mathbf x'^TF\mathbf x=0\) | Correspondence geometry |
| Essential matrix | \(E=[t]_\times R\) | Calibrated relative geometry |
| F–E relation | \(F=K'^{-T}EK^{-1}\) | Pixel versus normalized coordinates |
| Eight-point system | \(A\mathbf f=0\) | Linear estimation of F |
| Rank constraint | \(\operatorname{rank}(F)=2\) | Valid fundamental matrix |
| Denormalization | \(F=T'^TF_nT\) | Return to original pixel coordinates |
| Triangulation | \(\tilde x\sim P\tilde X\), \(\tilde x'\sim P'\tilde X\) | Recover 3D from two views |

---

# 12. Suggested lecture delivery and revision plan

**First 20 minutes — Build intuition.** Use the perspective ambiguity and epipolar diagrams (Figures 1–3). Ask students why two eyes are useful and why a match lies on a line rather than anywhere in the second image.

**Next 30 minutes — Do the geometry.** Explain rectified cameras, disparity, the depth equation, and error growth (Figures 4–6). Have students calculate two depths by hand.

**Next 25 minutes — Solve the correspondence problem.** Demonstrate SSD, disparity maps, and SIFT (Figures 7–9). Emphasize why visual similarity and geometric constraints complement each other.

**Next 35 minutes — Introduce F and E.** Start from the equation of a 2D line, then derive the epipolar constraint and compare pixel versus normalized coordinates (Figures 10–11).

**Next 30 minutes — Estimate F robustly.** Explain eight correspondences, SVD, rank-2 correction, normalization, and RANSAC (Figures 12–14).

**Final 20 minutes — Reconstruct 3D and recap.** Discuss triangulation, reprojection error, the full stereo pipeline, and practice questions.

---

## Source and attribution

**Primary source:** Supplied course handout, *AI6121 Computer Vision — 11. Epipolar & Stereo Vision* (2026), 58 pages. The lecture's original progression is retained: introduction (slides 3–8), parallel stereo and matching (slides 9–19), fundamental and essential matrices (slides 20–32), fundamental-matrix estimation (slides 33–52), and 3D reconstruction (slides 53–56, with supplementary slides following).

**Additional teaching material in these notes:** Beginner analogies, step-by-step derivations, numerical examples, conceptual clarifications, notation standardization, practical pipeline, misconceptions, exercises, and worked answers. These additions explain and contextualize the source rather than claiming to be verbatim slide content.

**Image use:** Figures are excerpts of the provided lecture slides for private educational study; copyright remains with the respective source owners.
