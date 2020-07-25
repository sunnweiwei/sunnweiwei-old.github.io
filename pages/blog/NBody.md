给定多星系统的初始状态，以一定的时间步，计算在引力作用下的星体运动，并使用openGL实时可视化。

## 实验环境

python37+OpenGL https://www.cnblogs.com/GraceSkyer/p/9235582.html

numpy

PIL

## 初始条件

使用一个数组p表示多星系统的初始条件

```
p = [[1.0, 0, 0, 0, 0, 0, 0],
     [1.0, 0, 10, 0, 0.2, 0, 0],
     [1.0, 10, 0, 0, 0, 0, 0.2],
     ]
```

数组每行表示一个星体的：质量，x坐标，y坐标，z坐标，x速度，y速度，z速度

单位分别是：kg，m，m，m，m/s，m/s，m/s

新建openGL运行类NBody，传入初始条件

```python
class NBody:
    def __init__(self, node, k, time_step, width=640, height=480, title='NBody'.encode()):
        self.ptcN = len(node)
        self.path = [deque() for _ in range(self.ptcN)]
        self.ptcM = node[:, 0]
        self.ptcP = node[:, 1:4]
        self.ptcV = node[:, 4:]
        self.step = time_step
        self.K = k
```

## 状态更新

计算每个node受到其他node的引力的合力，并以此计算node在时间步内的位置变化。

```python
for item in range(self.ptcN):
    force = np.array([0.0, 0, 0])
    for i in range(self.ptcN):
        if i == item:
           continue
        distance = self.ptcP[item] - self.ptcP[i]
        ss = 1/(sum(np.square(distance))**0.5+0.01)
        force -= ss**3*self.ptcM[i]*self.K*distance
    self.ptcV[item] += force
self.ptcP += self.ptcV*self.step
```

## 可视化

初始化

```python
self.color = []
for i in range(self.ptcN):
    self.color.append(np.random.randint(min(2 * i % 6, 2 * (i + 1) % 6), max(2 * i % 6, 2 * (i + 1) % 6)))
self.run = 1
glutInit(sys.argv)
glutInitDisplayMode(GLUT_RGBA | GLUT_DOUBLE | GLUT_DEPTH)
glutInitWindowSize(width, height)
self.window = glutCreateWindow(title)
glutDisplayFunc(self.Draw)
glutMouseFunc(self.myMouse)
glutMotionFunc(self.onMouseMove)
glutSpecialFunc(self.mySpecial)
glutIdleFunc(self.Draw)
self.InitGL(width, height)
self.xR = 0.0
self.yR = 0.0
self.Oldx = 0
self.Oldy = 0
self.scale = 1.0
self.tz = 50
```

绘制画布，先计算新的状态，再根据进行视角旋转，再绘制每个node，记录node位置的历史数据，根据历史数据在绘制运动轨迹。

```python
def Draw(self):
    if self.run:
        self.refresh()
    glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT)
    glLoadIdentity()
    glEnable(GL_DEPTH_TEST)
    glDepthFunc(GL_LEQUAL)
    glHint(GL_PERSPECTIVE_CORRECTION_HINT, GL_NICEST)

    # 沿z轴平移
    gluLookAt(0, 0, -self.tz,  # 相机在世界坐标系中的位置坐标
              0, 0, 0,  # 相机正对着的世界坐标系中的点的位置坐标，成像后这一点会位于画板的中心位置
              0, 1, 0)  # 定义相机本身的朝向
    # 分别绕x,y,z轴旋转
    glRotatef(self.xR, 1.0, 0.0, 0.0)
    glRotatef(self.yR, 0.0, 1.0, 0.0)
    glScalef(self.scale, self.scale, self.scale)
    glPushMatrix()
    for i, (x, y, z) in enumerate(self.ptcP):
        glBindTexture(GL_TEXTURE_2D, self.color[i])
        glTranslatef(x, y, z)
        glutSolidSphere((self.ptcM[i] + 1) ** 0.3, 10, 10)
        glTranslatef(-x, -y, -z)
        if np.random.randint(10) == 3:
            self.path[i].append([x, y, z])
        if len(self.path[i]) > 40:
            self.path[i].popleft()
        glBegin(GL_LINES)
        for t in range(len(self.path[i]) - 1):
            glVertex3f(self.path[i][t][0], self.path[i][t][1], self.path[i][t][2])
            glVertex3f(self.path[i][t + 1][0], self.path[i][t + 1][1], self.path[i][t + 1][2])
        glEnd()

    glPopMatrix()
    glFlush()

    # 刷新屏幕，产生动画效果
    glutSwapBuffers()
    # time.sleep(0.001)
```

鼠标控制，实现鼠标拖动时视角旋转。

```python
def myMouse(self, btn, state, x, y):
    if btn == GLUT_LEFT_BUTTON and state == GLUT_DOWN:
        self.Oldx, self.Oldy = x, y
    if state == GLUT_UP and btn == 3:
        self.scale += 1
    if state == GLUT_UP and btn == 4:
        self.scale -= 1
    glutPostRedisplay()

def onMouseMove(self, x, y):
    self.yR += x - self.Oldx
    self.yR = min(self.yR, 180.0)
    self.yR = max(self.yR, -180.0)
    glutPostRedisplay()
    self.Oldx = x
    self.xR -= y - self.Oldy
    self.xR = min(self.xR, 180.0)
    self.xR = max(self.xR, -180.0)
    glutPostRedisplay()
    self.Oldy = y
```

键盘控制，实现视角缩放和暂停

```python
def mySpecial(self, key, x, y):
    if key == GLUT_KEY_UP:
        self.tz -= 1
    if key == GLUT_KEY_DOWN:
        self.tz += 1
    if key == GLUT_KEY_F4:
        self.run = 1 - self.run
    glutPostRedisplay()

def myIdle(self):
    pass
```

加载纹理，每个node的纹理使用jpg纹理文件加载

```python
def LoadTexture(self):
    imgFiles = ['./color/' + str(i) + '.jpg' for i in range(6)]
    for i in range(6):
        img = Image.open(imgFiles[i])
        width, height = img.size
        img = img.tobytes('raw', 'RGBX', 0, -1)

        glGenTextures(2)
        glBindTexture(GL_TEXTURE_2D, i)
        glTexImage2D(GL_TEXTURE_2D, 0, 4,
                     width, height, 0, GL_RGBA,
                     GL_UNSIGNED_BYTE, img)
        glTexParameterf(GL_TEXTURE_2D,
                        GL_TEXTURE_WRAP_S, GL_CLAMP)
        glTexParameterf(GL_TEXTURE_2D,
                        GL_TEXTURE_WRAP_T, GL_CLAMP)
        glTexParameterf(GL_TEXTURE_2D,
                        GL_TEXTURE_WRAP_S, GL_REPEAT)
        glTexParameterf(GL_TEXTURE_2D,
                        GL_TEXTURE_WRAP_T, GL_REPEAT)
        glTexParameterf(GL_TEXTURE_2D,
                        GL_TEXTURE_MAG_FILTER, GL_NEAREST)
        glTexParameterf(GL_TEXTURE_2D,
                        GL_TEXTURE_MIN_FILTER, GL_NEAREST)
        glTexEnvf(GL_TEXTURE_ENV,
                  GL_TEXTURE_ENV_MODE, GL_DECAL)
```

设置InitGL

```python
def InitGL(self, width, height):
    self.LoadTexture()
    glEnable(GL_TEXTURE_2D)
    glClearColor(1.0, 1.0, 1.0, 0.0)
    glClearDepth(1.0)
    glDepthFunc(GL_LESS)
    glShadeModel(GL_SMOOTH)
    # 背面剔除，消隐

    glEnable(GL_CULL_FACE)
    glCullFace(GL_BACK)

    glEnable(GL_POINT_SMOOTH)
    glEnable(GL_LINE_SMOOTH)
    glEnable(GL_POLYGON_SMOOTH)
    glMatrixMode(GL_PROJECTION)
    glLoadIdentity()
    gluPerspective(45.0, float(width) / float(height), 0.1, 100.0)
    glMatrixMode(GL_MODELVIEW)
```

## 运行效果



可实现功能：

- 对N体问题进行模拟
- 随机选择预设的纹理，进行可视化展示
- 使用鼠标控制视角变化
- 对历史轨迹可视化



## 总结

基于openGL实现的N体问题模拟。在低速，较少节点下有很好的运行效果。如果节点较多可以考虑使用八叉树进行优化。基本实现了openGL的基本图形显示，视角控制和简单纹理功能。

三体问题（three-body problem）是天体力学中的基本力学模型。它是指三个质量、初始位置和初始速度都是任意的可视为质点的天体，在相互之间万有引力的作用下的运动规律问题。现已知三体问题不能精确求解，即无法预测所有三体问题的数学情景，只有几种特殊情况已研究。三体问题最简单的一个例子就是太阳系中太阳、地球和月球的运动。在浩瀚的宇宙中，星球的大小可以忽略不记，所以我们可以把它们看成质点。如果不计太阳系其他星球的影响，那么它们的运动就只是在引力的作用下产生的，所以我们就可以把它们的运动看成一个三体问题。研究三体问题的方法大致可分为分析方法、定性方法、数值方法三类。

我看到了我的爱恋
我飞到她的身边
我捧出给她的礼物
那是一小块凝固的时间
时间上有美丽的条纹
摸起来像浅海的泥一样柔软

她把时间涂满全身
然后拉起我飞向存在的边缘
这是灵态的飞行
我们眼中的星星像幽灵
星星眼中的我们也像幽灵



## 完整代码

```python
import sys
from OpenGL.GL import *
from OpenGL.GLUT import *
from OpenGL.GLU import *
from PIL import Image
import numpy as np
from collections import deque
import time


class NBody:
    def __init__(self, node, k, time_step, width=640, height=480, title='NBody'.encode()):
        self.ptcN = len(node)
        self.path = [deque() for _ in range(self.ptcN)]
        self.ptcM = node[:, 0]
        self.ptcP = node[:, 1:4]
        self.ptcV = node[:, 4:]
        self.step = time_step
        self.K = k
        self.color = []
        for i in range(self.ptcN):
            self.color.append(np.random.randint(min(2 * i % 6, 2 * (i + 1) % 6), max(2 * i % 6, 2 * (i + 1) % 6)))
        self.run = 1
        glutInit(sys.argv)
        glutInitDisplayMode(GLUT_RGBA | GLUT_DOUBLE | GLUT_DEPTH)
        glutInitWindowSize(width, height)
        self.window = glutCreateWindow(title)
        glutDisplayFunc(self.Draw)
        glutMouseFunc(self.myMouse)
        glutMotionFunc(self.onMouseMove)
        glutSpecialFunc(self.mySpecial)
        glutIdleFunc(self.Draw)
        self.InitGL(width, height)
        self.xR = 0.0
        self.yR = 0.0
        self.Oldx = 0
        self.Oldy = 0
        self.scale = 1.0
        self.tz = 50

    def refresh(self):
        for item in range(self.ptcN):
            force = np.array([0.0, 0, 0])
            for i in range(self.ptcN):
                if i == item:
                    continue
                distance = self.ptcP[item] - self.ptcP[i]
                ss = 1 / (sum(np.square(distance)) ** 0.5 + 0.001)
                force -= ss ** 3 * self.ptcM[i] * self.K * distance
            self.ptcV[item] += force
        self.ptcP += self.ptcV * self.step

    def setup(self):
        glClearColor(1.0, 1.0, 1.0, 1.0)
        glShadeModel(GL_SMOOTH)
        ambient = [0.4, 0.6, 0.2, 1.0]
        position = [1.0, 1.0, 3.0, 1.0]
        glEnable(GL_LIGHTING)
        glEnable(GL_LIGHT0)
        glLightfv(GL_LIGHT0, GL_AMBIENT, ambient)
        glLightfv(GL_LIGHT0, GL_POSITION, position)

        mat_ambient = [0.5, 0.2, 0.5, 1.0]
        mat_diffuse = [0.8, 0.6, 0.3, 1.0]
        mat_specular = [0.8, 0.6, 0.3, 1.0]
        mat_shininess = [45.0]

        glMaterialfv(GL_FRONT, GL_AMBIENT, mat_ambient)
        glMaterialfv(GL_FRONT, GL_DIFFUSE, mat_diffuse)
        glMaterialfv(GL_FRONT, GL_SPECULAR, mat_specular)
        glMaterialfv(GL_FRONT, GL_SHININESS, mat_shininess)

        glEnable(GL_COLOR_MATERIAL)
        glColorMaterial(GL_FRONT, GL_AMBIENT_AND_DIFFUSE)
        glDepthFunc(GL_LESS)
        glEnable(GL_DEPTH_TEST)
        glEnable(GL_AUTO_NORMAL)
        glEnable(GL_NORMALIZE)

    # 绘制图形
    def Draw(self):
        if self.run:
            self.refresh()
        glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT)
        glLoadIdentity()
        glEnable(GL_DEPTH_TEST)
        glDepthFunc(GL_LEQUAL)
        glHint(GL_PERSPECTIVE_CORRECTION_HINT, GL_NICEST)

        # 沿z轴平移
        gluLookAt(0, 0, -self.tz,  # 相机在世界坐标系中的位置坐标
                  0, 0, 0,  # 相机正对着的世界坐标系中的点的位置坐标，成像后这一点会位于画板的中心位置
                  0, 1, 0)  # 定义相机本身的朝向
        # 分别绕x,y,z轴旋转
        glRotatef(self.xR, 1.0, 0.0, 0.0)
        glRotatef(self.yR, 0.0, 1.0, 0.0)
        glScalef(self.scale, self.scale, self.scale)
        glPushMatrix()
        for i, (x, y, z) in enumerate(self.ptcP):
            glBindTexture(GL_TEXTURE_2D, self.color[i])
            glTranslatef(x, y, z)
            glutSolidSphere((self.ptcM[i] + 1) ** 0.3, 10, 10)
            glTranslatef(-x, -y, -z)
            if np.random.randint(10) == 3:
                self.path[i].append([x, y, z])
            if len(self.path[i]) > 40:
                self.path[i].popleft()
            glBegin(GL_LINES)
            for t in range(len(self.path[i]) - 1):
                glVertex3f(self.path[i][t][0], self.path[i][t][1], self.path[i][t][2])
                glVertex3f(self.path[i][t + 1][0], self.path[i][t + 1][1], self.path[i][t + 1][2])
            glEnd()

        glPopMatrix()
        glFlush()

        # 刷新屏幕，产生动画效果
        glutSwapBuffers()
        # time.sleep(0.001)

    def myMouse(self, btn, state, x, y):
        if btn == GLUT_LEFT_BUTTON and state == GLUT_DOWN:
            self.Oldx, self.Oldy = x, y
        if state == GLUT_UP and btn == 3:
            self.scale += 1
        if state == GLUT_UP and btn == 4:
            self.scale -= 1
        glutPostRedisplay()

    def onMouseMove(self, x, y):
        self.yR += x - self.Oldx
        self.yR = min(self.yR, 180.0)
        self.yR = max(self.yR, -180.0)
        glutPostRedisplay()
        self.Oldx = x
        self.xR -= y - self.Oldy
        self.xR = min(self.xR, 180.0)
        self.xR = max(self.xR, -180.0)
        glutPostRedisplay()
        self.Oldy = y

    def mySpecial(self, key, x, y):
        if key == GLUT_KEY_UP:
            self.tz -= 1
        if key == GLUT_KEY_DOWN:
            self.tz += 1
        if key == GLUT_KEY_F4:
            self.run = 1 - self.run
        glutPostRedisplay()

    def myIdle(self):
        pass

    def LoadTexture(self):
        # 提前准备好的6个图片
        imgFiles = ['./color/' + str(i) + '.jpg' for i in range(6)]
        for i in range(6):
            img = Image.open(imgFiles[i])
            width, height = img.size
            img = img.tobytes('raw', 'RGBX', 0, -1)

            glGenTextures(2)
            glBindTexture(GL_TEXTURE_2D, i)
            glTexImage2D(GL_TEXTURE_2D, 0, 4,
                         width, height, 0, GL_RGBA,
                         GL_UNSIGNED_BYTE, img)
            glTexParameterf(GL_TEXTURE_2D,
                            GL_TEXTURE_WRAP_S, GL_CLAMP)
            glTexParameterf(GL_TEXTURE_2D,
                            GL_TEXTURE_WRAP_T, GL_CLAMP)
            glTexParameterf(GL_TEXTURE_2D,
                            GL_TEXTURE_WRAP_S, GL_REPEAT)
            glTexParameterf(GL_TEXTURE_2D,
                            GL_TEXTURE_WRAP_T, GL_REPEAT)
            glTexParameterf(GL_TEXTURE_2D,
                            GL_TEXTURE_MAG_FILTER, GL_NEAREST)
            glTexParameterf(GL_TEXTURE_2D,
                            GL_TEXTURE_MIN_FILTER, GL_NEAREST)
            glTexEnvf(GL_TEXTURE_ENV,
                      GL_TEXTURE_ENV_MODE, GL_DECAL)

    def InitGL(self, width, height):
        self.LoadTexture()
        glEnable(GL_TEXTURE_2D)
        glClearColor(1.0, 1.0, 1.0, 0.0)
        glClearDepth(1.0)
        glDepthFunc(GL_LESS)
        glShadeModel(GL_SMOOTH)
        # 背面剔除，消隐

        glEnable(GL_CULL_FACE)
        glCullFace(GL_BACK)

        glEnable(GL_POINT_SMOOTH)
        glEnable(GL_LINE_SMOOTH)
        glEnable(GL_POLYGON_SMOOTH)
        glMatrixMode(GL_PROJECTION)
        glLoadIdentity()
        gluPerspective(45.0, float(width) / float(height), 0.1, 100.0)
        glMatrixMode(GL_MODELVIEW)

    def MainLoop(self):
        glutMainLoop()


if __name__ == '__main__':
    # weight, position_x, y, z, velocity_x, y, z
    p = [[1.0, 0, 0, 0, 0, 0, 0],
         [0, 0, 10, 0, 0.2, 0, 0],
         [0, 10, 0, 0, 0, 0, 0.2],
         [0, 0, 0, 10, 0, 0.2, 0],
         [0, 0, -10, 0, -0.2, 0, 0],
         [0, -10, 0, 0, 0, 0, -0.2],
         [0, 0, 0, -10, 0, -0.2, 0],
         ]

    p = np.array(p)
    w = NBody(p, 0.1, 0.1)
    w.MainLoop()

```

