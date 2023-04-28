# Hazel引擎学习记录

### 添加ImGui

bug:ImGui显示不完全

原因：WindowsWindow.h文件中内联函数为

```c++
inline unsigned int GetWidth() const override { return m_Data.height; }
```

应修改为

```c++
inline unsigned int GetWidth() const override { return m_Data.Width; }
```

### 为ImGui添加事件

ImGui不响应事件

原因：事件函数写法逻辑出现问题，released函数中写成pressed的逻辑

### 添加数学库glm

bug:链接错误，找不到s_Instance的符号

原因：WindowsInput类的文件未被包含至正确文件夹下

### 调整Hazel为静态库

```shell
warning C4251: 'Hazel::Application::m_Window': class 'std::unique_ptr<Hazel::Window,std::default_delete<_Ty>>' needs to have dll-interface to be used by clients of class 'Hazel::Application
```

以上错误是由于dll使用的编译器与作dll的Application实际使用的编译器不保证相同,使用dll的Application有权限访问数据成员，就会发出警告

### Rendering渲染

Render context：与平台相关，不同的平台对应的Render context不一

### Our first triangle

首次渲染三角形时，以下代码报错**C2338 static_assert failed: 'Formatting of non-void pointers is disallowed.'**

```c++
HZ_CORE_INFO("  Renderer: {0}", glGetString(GL_RENDERER));
HZ_CORE_INFO("  Version: {0}", glGetString(GL_VERSION));
```

原因：spdlog在使用fmt格式化库运行时需要格式化参数,使用fmt::ptr()函数解决

参考链接：[How to output Pointer · Issue #1492 · gabime/spdlog (github.com)](https://github.com/gabime/spdlog/issues/1492)

初步解决方案如下

```C++
HZ_CORE_INFO("  Renderer: {0}", fmt::ptr(glGetString(GL_RENDERER)));
HZ_CORE_INFO("  Version: {0}", fmt::ptr(glGetString(GL_VERSION)));
```

缺陷：只能返回指针地址

最终解决方案：使用(const char*)转换string到c的字符型

参考链接：[How to output pointers in spdlog · Issue #621 · TheCherno/Hazel (github.com)](https://github.com/TheCherno/Hazel/issues/621)

```C++
HZ_CORE_INFO("  Renderer: {0}", (const char*)glGetString(GL_RENDERER));
HZ_CORE_INFO("  Version: {0}", (const char*)glGetString(GL_VERSION));
```

注：[(1条消息) 【现代C++】新的字符串格式化方法_tangclfs的博客-CSDN博客](https://blog.csdn.net/qq_17291647/article/details/117376889)

### Our first Shader

##### GLSL

为图形计算定制的类C语言

##### 输入输出

使用关键字in,out，进行着色器间的数据交流，使用location这一元数据指定输入变量

注：unique_ptr的u.reset(q)函数为释放u原来指向的对象，令u指向这个q

### buffer布局

注：C++11特性，列表初始化initializer_list，对自定义类的初始化

注：顶点数组对象(Vertex Array Object,VAO)绑定之后，任何的顶点属性调用都存储在这个VAO中，当配置顶点属性指针时候，只需要执行调用一次，再次绘制只需要绑定响应的VAO即可

![img](https://learnopengl-cn.github.io/img/01/04/vertex_array_objects.png)
