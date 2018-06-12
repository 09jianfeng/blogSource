---
title: OpenGL全景播放
date: 2017-01-08 10:47:41
tags: [openGL]
---

```
#pragma mark - 渲染
- (void)update {
    float aspect = fabs(self.bounds.size.width / self.bounds.size.height);
    
    //透视矩阵
    GLKMatrix4 projectionMatrix = GLKMatrix4MakePerspective(GLKMathDegreesToRadians(self.overture), aspect, 0.1f, 400.0f);
    
    //model矩阵
    GLKMatrix4 modelViewMatrix = GLKMatrix4Identity;
    modelViewMatrix = GLKMatrix4Scale(modelViewMatrix, 300.0, 300.0, 300.0);
    
    GLKMatrix4 lookatMatrix = GLKMatrix4MakeLookAt(0, 0.0, 1.0, 0, 0, 0, 0, 1, 0);
    _isLandscape = YES;
    if (_isUsingMotion) {
        if (_isLandscape) {
            //顶点物体本身的x,y,z三个轴。
            //注意： x,y,z是对应于承载这个opengl es层的那个view的x,y,z。竖屏x,y,z就是跟屏幕的x,y,z轴一致。如果是横屏，则x轴与y轴跟屏幕的是对调的。 opengl这里的model的坐标参考系是跟view的一致的。
            modelViewMatrix = GLKMatrix4Rotate(modelViewMatrix, -M_PI_2, 1.0f, 0.0f, 0.0f);
            CMRotationMatrix a = _mineattitude.rotationMatrix;
            GLKMatrix4 rotatoinMatri = GLKMatrix4Make(a.m11, a.m21, a.m31, 0.0f,
                                                      a.m12, a.m22, a.m32, 0.0f,
                                                      a.m13, a.m23, a.m33, 0.0f,
                                                      0.0f,  0.0f,  0.0f,  1.0f);
            //手机平放在桌面，陀螺仪的方向是对应着lookat矩阵代表的摄像头方向是只向Z方向。（也就是说把Lookat矩阵的开始方向要对着Z的方向），并且摄像机Up方向是Y正方向。
            rotatoinMatri = GLKMatrix4RotateZ(rotatoinMatri, M_PI_2);
            
            lookatMatrix = GLKMatrix4Multiply(lookatMatrix, rotatoinMatri);
            //look at的坐标系就是手机y坐标方向是Y轴，x坐标方向是x轴，正对着手机的Z轴。
            //注意：判断坐标系的时候要从模型最开始的状态，紧跟整个模型/摄像机在3D空间中的方向，然后再考虑摄像机应该绕着哪条轴旋转。不要被从手机看到的画面迷惑。 提示：可以把手机想象成整个3D模型中的一个平面，竖直放着手机，竖直方向是Y轴，横向是X轴，垂直于手机屏的是Z轴
            lookatMatrix = GLKMatrix4RotateX(lookatMatrix, self.fingerRotationX);
            //这里在屏幕上滑动的左右滑，为什么却是围绕着Z轴旋转？ 想象一下上面的注释，提示：抬起手机的时候lookat矩阵已经是指向了的球体的顶部（陀螺仪的旋转矩阵导致的）。
            lookatMatrix = GLKMatrix4RotateZ(lookatMatrix, -self.fingerRotationY);
            
        }else{
            
            {
                //顶点物体本身的x,y,z三个轴。
                //注意： x,y,z是对应于承载这个opengl es层的那个view的x,y,z。竖屏x,y,z就是跟屏幕的x,y,z轴一致。如果是横屏，则x轴与y轴跟屏幕的是对调的。 opengl这里的model的坐标参考系是跟view的一致的。
                modelViewMatrix = GLKMatrix4Rotate(modelViewMatrix, -M_PI_2, 1.0f, 0.0f, 0.0f);
                CMRotationMatrix a = _mineattitude.rotationMatrix;
                GLKMatrix4 rotatoinMatri = GLKMatrix4Make(a.m11, a.m21, a.m31, 0.0f,
                                                          a.m12, a.m22, a.m32, 0.0f,
                                                          a.m13, a.m23, a.m33, 0.0f,
                                                          0.0f,  0.0f,  0.0f,  1.0f);
                //手机平放在桌面，陀螺仪的方向是对应着lookat矩阵代表的摄像头方向是只向Z方向。（也就是说把Lookat矩阵的开始方向要对着Z的方向），并且摄像机Up方向是Y正方向。
                lookatMatrix = GLKMatrix4Multiply(lookatMatrix, rotatoinMatri);
                //look at的坐标系就是手机y坐标方向是Y轴，x坐标方向是x轴，正对着手机的Z轴。
                //注意：判断坐标系的时候要从模型最开始的状态，紧跟整个模型/摄像机在3D空间中的方向，然后再考虑摄像机应该绕着哪条轴旋转。不要被从手机看到的画面迷惑。 提示：可以把手机想象成整个3D模型中的一个平面，竖直放着手机，竖直方向是Y轴，横向是X轴，垂直于手机屏的是Z轴
                lookatMatrix = GLKMatrix4RotateX(lookatMatrix, self.fingerRotationX);
                //这里在屏幕上滑动的左右滑，为什么却是围绕着Z轴旋转？ 想象一下上面的注释，提示：抬起手机的时候lookat矩阵已经是指向了的球体的顶部（陀螺仪的旋转矩阵导致的）。
                lookatMatrix = GLKMatrix4RotateZ(lookatMatrix, -self.fingerRotationY);
            }
            
            /*
            {   //这个通过旋转LookAt矩阵得到的结果跟上面的先通过旋转 model的结果一样。只是上面的比较好理解
                CMRotationMatrix a = _mineattitude.rotationMatrix;
                GLKMatrix4 rotatoinMatri = GLKMatrix4Make(a.m11, a.m21, a.m31, 0.0f,
                                                          a.m12, a.m22, a.m32, 0.0f,
                                                          a.m13, a.m23, a.m33, 0.0f,
                                                          0.0f,  0.0f,  0.0f,  1.0f);
                lookatMatrix = GLKMatrix4Multiply(lookatMatrix, rotatoinMatri);
                lookatMatrix = GLKMatrix4Rotate(lookatMatrix, -M_PI_2, 1.0f, 0.0f, 0.0f);
                lookatMatrix = GLKMatrix4RotateX(lookatMatrix, self.fingerRotationX);
                lookatMatrix = GLKMatrix4RotateY(lookatMatrix, self.fingerRotationY);
            }
             */
        }
    }else{
        lookatMatrix = GLKMatrix4RotateX(lookatMatrix, self.fingerRotationX);
        lookatMatrix = GLKMatrix4RotateY(lookatMatrix, -self.fingerRotationY);
    }
    
    self.glkView.frame = self.frame;
    self.modelViewProjectionMatrix = GLKMatrix4Multiply(projectionMatrix, lookatMatrix);
    self.modelViewProjectionMatrix = GLKMatrix4Multiply(self.modelViewProjectionMatrix, modelViewMatrix);
}

```