图形学中的矩阵变换
# 摘要
图形学因为要处理三维中的物体，所以经常要用到矩阵变换，包括基础的模型变换（旋转、平移、缩放变换），以及投影变换、视口变换。这其中有很多有意思的数学知识。网络上虽然有很多介绍矩阵变换的博客，但是很多只介绍了基本的矩阵运算，没有深入介绍四元数等高级用法。因此本文将从模型变换，特别是旋转变换切入，介绍四元数、矩阵分解操作等内容。
# 模型变换
模型变换是将一个点（模型）变换到另一个点（模型）的操作。一般我们会把点用齐次坐标（homogeneous notation）表示，所以变换矩阵是$4\times4$的。另一个需要注意的点是，矩阵乘法运算一般情况下不满足交换律，所以变换的顺序很重要，一个直观的例子是原点处的物体先平移再绕原点旋转的结果与先绕原点旋转再平移的结果是不一样的。这里我们约定，平移变换$T$、缩放变换$S$、旋转变换$R$的顺序依次是先缩放再旋转最后平移，表示为：
$$
M=TRS
$$
## 平移变换
平移变换比较自然的一种表示形式是向量——这也是向量的物理意义。假设一个平移操作${\bf{t}}=(t_x, t_y, t_x)$，写为矩阵是：
$$
{\bf{T}}({\bf{t}}) = T(t_x, t_y, t_z) = 
\begin{pmatrix}
    1 & 0 & 0 & t_x \\
    0 & 1 & 0 & t_y \\
    0 & 0 & 1 & t_x \\
    0 & 0 & 0 & 1 \\
\end{pmatrix}
$$
平移矩阵的逆矩阵也很容易求，将${\bf{t}}$取负即可：${\bf{T}}^{-1}({\bf{t}})={\bf{T}}(-{\bf{t}})$。逻辑上比较容易理解。
## 缩放变换
缩放变换用于将$x$、$y$、$z$轴依次按系数$(s_x, s_y, s_y)$进行缩放，缩放矩阵表示为：
$$
{\bf{S}}({\bf{s}}) =
\begin{pmatrix}
    s_x & 0 & 0 & 0 \\
    0 & s_y & 0 & 0 \\
    0 & 0 & s_z & 0 \\
    0 & 0 & 0 & 1 \\
\end{pmatrix}
$$
其逆矩阵${\bf{S}}^{-1}({\bf{s}})={\bf{S}}(1/s_x, 1/s_y, 1/s_z)$。
### 沿任意轴的缩放
TODO
## 旋转变换
旋转变换相较于前两种变换要复杂一些。我们先看简单的形式，假设旋转轴是$x$、$y$、$z$轴，旋转角度为$\phi$，那么旋转矩阵可以写为：
$$
{\bf{R}}_x({\bf{\phi}}) =
\begin{pmatrix}
    1 & 0 & 0 & 0 \\
    0 & \cos\phi & -\sin\phi & 0 \\
    0 & \sin\phi & \cos\phi & 0 \\
    0 & 0 & 0 & 1 \\
\end{pmatrix} \\
{\bf{R}}_y({\bf{\phi}}) =
\begin{pmatrix}
    \cos\phi & 0 & \sin\phi & 0 \\
    0 & 1 & 0 & 0 \\
    -\sin\phi & 0 & \cos\phi & 0 \\
    0 & 0 & 0 & 1 \\
\end{pmatrix} \\
{\bf{R}}_z({\bf{\phi}}) =
\begin{pmatrix}
    \cos\phi & -\sin\phi & 0 & 0 \\
    \sin\phi & \cos\phi & 0 & 0 \\
    0 & 0 & 1 & 0 \\
    0 & 0 & 0 & 1 \\
\end{pmatrix}
$$
其逆矩阵
$$
{\bf{R}}^{-1}_i(\phi)={\bf{R}}_i(-\phi)={\bf{R}}^{T}_i(\phi)
$$
其他一些有趣的性质包括：
$$
\begin{aligned}
{\bf{R}}_i(0) & = {\bf{I}} \\
{\bf{R}}_i(\phi_1){\bf{R}}_i(\phi_2) & = {\bf{R}}_i(\phi_1 + \phi_2) \\
{\bf{R}}_i(\phi_1){\bf{R}}_i(\phi_2) & = {\bf{R}}_i(\phi_2){\bf{R}}_i(\phi_1)
\end{aligned}
$$
### 绕任意轴的旋转
绕任意轴的旋转看起来会比较复杂，但是仔细分析一下其实并不困难。推导过程主要分两步，

1. 求旋转前后两个向量的关系；
2. 将这个关系转换为矩阵；

假设我们需要将向量${\bf{v}}$绕着轴${\bf{a}}$旋转角度$\theta$，如下图所示。

![Rotate arbitrary axis](./images/Rotate_arbitrary_axis.png)

令${\bf{v}}_c$表示${\bf{v}}$在轴${\bf{a}}$上的投影，即：
$$
{\bf{v}}_c = {\bf{a}}||{\bf{v}}||\cos\alpha={\bf{a}}({\bf{v}}\cdot{\bf{a}})
$$
并且令：
$$
{\bf{v}}_1={\bf{v}}-{\bf{v}}_c \\ 
{\bf{v}}_2={\bf{v}}_1\times{\bf{a}}
$$
则根据${\bf{v}}_1$与${\bf{v}}_1'$、${\bf{v}}_2$与${\bf{v}}_2'$的关系，可以求得：
$$
{\bf{v}}'={\bf{v}}_c+{\bf{v}}_1\cos\theta+{\bf{v}}_2\sin\theta
$$
这样就得到了${\bf{v}}'$与${\bf{v}}$的关系。
接下来，

```C++
Matrix4x4 Rotate(Float theta, const Vector3f &axis) {
    Vector3f a = Normalize(axis);
    Float sinTheta = std::sin(Radians(theta));
    Float cosTheta = std::cos(Radians(theta));
    Matrix4x4 m;
    // Compute rotation of first basis vector
    m.m[0][0] = a.x * a.x + (1 - a.x * a.x) * cosTheta;
    m.m[0][1] = a.x * a.y * (1 - cosTheta) - a.z * sinTheta;
    m.m[0][2] = a.x * a.z * (1 - cosTheta) + a.y * sinTheta;
    m.m[0][3] = 0;

    // Compute rotations of second and third basis vectors
    m.m[1][0] = a.x * a.y * (1 - cosTheta) + a.z * sinTheta;
    m.m[1][1] = a.y * a.y + (1 - a.y * a.y) * cosTheta;
    m.m[1][2] = a.y * a.z * (1 - cosTheta) - a.x * sinTheta;
    m.m[1][3] = 0;

    m.m[2][0] = a.x * a.z * (1 - cosTheta) - a.y * sinTheta;
    m.m[2][1] = a.y * a.z * (1 - cosTheta) + a.x * sinTheta;
    m.m[2][2] = a.z * a.z + (1 - a.z * a.z) * cosTheta;
    m.m[2][3] = 0;
    return m;
}
```

### 欧拉角与旋转变换

# 四元数法

# 法向变换
