# 说说单应性矩阵(homography)在拼接全景图中的应用

单应性矩阵为一3x3矩阵，描述了射影几何中平面到平面的映射关系，其自由度为8，由九个元素组成，通常令最后一个元素为1或者使其F范数为1，该矩阵可将无穷远点投射于有限处，即空间中平行线在图像上相交于有限处。

$$
s \begin{bmatrix} x’ \\ y’ \\ 1 \end{bmatrix} = H \begin{bmatrix} x \\ y \\ 1 \end{bmatrix} = \begin{bmatrix} h_{11} & h_{12} & h_{13} \\ h_{21} & h_{22} & h_{23} \\ h_{31} & h_{32} & h_{33} \end{bmatrix} \begin{bmatrix} x \\ y \\ 1 \end{bmatrix}  \tag{1}
$$

若最后一行为 $\bigl( \begin{smallmatrix} 0 & 0 & 1 \end{smallmatrix} \bigr)$，则不能将无穷远点投射于有限处，称为仿射变换（Affine），自由度为6，如下：
$$
A= \begin{bmatrix}
   *  & * & * \\
   *  & * & * \\
   0 & 0 & 1
  \end{bmatrix} \tag{2}
$$

此外还有4自由度的相似变换和3自由度的等距变换.。

----------------------------------
-  **全景图拼接**

理论上要求用于拼接的图片拍摄时其相机光心需要重合。

![全景图拼接示意图](http://img.blog.csdn.net/20180117094958620?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSjEwNTI3/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

通常用于拼接全景图的照片都是拍摄的大场景，可以近似认为其拍摄相机光心重合，若各照片拍摄时光心偏离越大拼接效果就会越差。

获取匹配点对过程略。

-  **由匹配点对估计变换矩阵**

图像对之间的运动假设只存在三维旋转，则变换为：
$$
s_{i}\begin{bmatrix} u'_{i} \\ v'_{i} \\ 1 \end{bmatrix}=H\begin{bmatrix} u_{i} \\ v_{i} \\ 1 \end{bmatrix}=K'RK^{-1}  \begin{bmatrix} u_{i} \\ v_{i} \\ 1 \end{bmatrix} \tag{3}
$$

其中$K$和$K'$为相机内参数矩阵，$R$为旋转矩阵，$H$为变换矩阵。若已知内参数矩阵，使用umeyama绝对定向方法只需要两个匹配点对即可估计出变换矩阵$H$。下面的代码展示了由两组匹配点对在相机坐标系下的齐次坐标，计算三维旋转的过程。

```cpp
/*
* in:  q1, q2 n-by-2 matrix, correspondance represented in the camera coordinate system, not in the image coordinate system
* out: R 3-by-3 rotation matrix
* author: Antonio Fan
* date:   Jan. 17th. 2018.
* email:  jah10527@126.com
* referance: "Least-squares estimation of transformation parameters between two point patterns", Shinji Umeyama, PAMI 1991, DOI: 10.1109/34.88573
*/
void umeyama(cv::Mat &q1, cv::Mat &q2, cv::Mat &R)
{
	auto p1 = q1.ptr<cv::Point2f>();
	auto p2 = q2.ptr<cv::Point2f>();

	cv::Matx33d A = cv::Matx33d::zeros();
	A(2, 2) = q1.rows;
	for (int i = 0; i < q1.rows; ++i)
	{
		A(0, 0) += p1[i].x*p2[i].x;
		A(0, 1) += p1[i].y*p2[i].x;
		A(0, 2) += p2[i].x;
		A(1, 0) += p1[i].x*p2[i].y;
		A(1, 1) += p1[i].y*p2[i].y;
		A(1, 2) += p2[i].y;
		A(2, 0) += p1[i].x;
		A(2, 1) += p1[i].y;
	}

	cv::Mat w, u, vt;
	cv::SVD::compute(A, w, u, vt);
	w.at<double>(0) = 1;
	w.at<double>(1) = 1;
	w.at<double>(2) = cv::determinant(u)*cv::determinant(vt);

	R = u*cv::Mat::diag(w)*vt;
}


int main()
{
	cv::RNG rnger(cv::getTickCount());
	int num = 2;
	cv::Mat pts3D(num, 3, CV_32FC1);
	rnger.fill(pts3D, cv::RNG::UNIFORM, cv::Scalar::all(-1.f), cv::Scalar::all(1.f));
	pts3D.col(2) += 2;
	
	cv::Mat R;
	cv::Matx31f rvec;
	rvec(0) = .1;
	rvec(1) = .01;
	rvec(2) = .01;
	cv::Rodrigues(rvec, R);
	std::cout << "R: " << R << std::endl;

	cv::Mat pts1, pts2;
	cv::convertPointsFromHomogeneous(pts3D, pts1);

	//std::cout << pts1 << std::endl;

	pts3D = R*pts3D.t();
	pts3D = pts3D.t();
	cv::convertPointsFromHomogeneous(pts3D, pts2);
	//std::cout << pts2 << std::endl;

	cv::Mat Restimated;
	umeyama(pts1, pts2, Restimated);
	std::cout << "Restimated: " << Restimated << std::endl;

	return 0;
}
```

该方法即是根据相机纯旋转情况下拍摄的图片来估计旋转的方法。如果既有旋转又有平移量，则应该使用本质矩阵（Essential matrix）来估计运动。

-------------------

### opencv中全景图的处理方法

opencv中的stitching module的H估计过程是，使用findHomography获取变换矩阵$H$作为初值进行光束平差。其中findHomography使用RANSAC与四个匹配点对最小二乘搭配求解$H$，获取初值后再进行全局重投影优化获取最终的变换矩阵系列。

