---
title: "Clion配置OpenGL学习环境——glfw+glad+assimp"
date: 2025-03-31
tags: ["OpenGL", "Clion", "C++"]
ShowToc: true
TocOpen: false
summary: "使用Clion配置OpenGL学习环境，包括glfw、glad、assimp库的完整配置流程，以及切换到VS工具链的详细教程。"
---

相信很多学习opengl的朋友都是从learnopengl这个教程开始的。网络上有很多配置opengl环境的教程，但大多数使用的ide都是vs，Clion相比vs有一个巨大的优势可以通过Cmakelists配置运行多个main函数，意味着在学习后面内容的章节时不用将以前写的代码清除掉，可以直接新开一个cpp文件写一个新的main函数。

<div style="display:flex;justify-content:center">
  <img src="/images/clion_setup/1.png"/>
</div>

由于learnopengl教程中使用的模型导入库assimp没有mingw的预编译版本，自己使用mingw编译也一直报错，所以会将Clion切换到vs工具链以便正常使用assimp库。

## 一、切换Clion到vs工具链

开始之前，确保已经安装好了vs的c++开发环境。

<div style="display:flex;justify-content:center">
  <img src="/images/clion_setup/2.png"/>
</div>

Clion中创建好C++项目后，来到设置->构建、执行、部署->工具链，点击添加一个工具链，工具集选择你的vs安装目录，下面的不用管会自动识别，点击应用。

<div style="display:flex;justify-content:center">
  <img src="/images/clion_setup/3.png"/>
</div>

之后来到设置->构建、执行、部署->Cmake，点击添加一个配置文件，取一个名字，就叫Debug-vs2022，可以把默认的mingw也重命名为Debug-Mingw方便区分。构建类型选择Debug，工具链选择刚刚添加的vs工具链，点击应用，确定。

<div style="display:flex;justify-content:center">
  <img src="/images/clion_setup/4.png"/>
</div>

之后右上角的cmake配置文件就会出现两个选项，左侧项目目录内也会出现两个配置分别对应的目录。使用vs的配置文件运行一下实例代码看看是否能正常运行。

<div style="display:flex;justify-content:center">
  <img src="/images/clion_setup/5.png"/>
</div>

之后，在项目根目录下创建include、lib、src三个目录。

<div style="display:flex;justify-content:center">
  <img src="/images/clion_setup/6.png"/>
</div>

## 二、配置glfw

到glfw官网 [An OpenGL library | GLFW](https://www.glfw.org/) 下载压缩包并解压，获得以下目录。

<div style="display:flex;justify-content:center">
  <img src="/images/clion_setup/7.png"/>
</div>

<div style="display:flex;justify-content:center">
  <img src="/images/clion_setup/8.png"/>
</div>

将include/GLFW整个文件夹放入include目录下，lib-vc2022/glfw3.lib放入lib目录下。

<div style="display:flex;justify-content:center">
  <img src="/images/clion_setup/9.png"/>
</div>

## 三、配置glad

到glad官网 [glad.dav1d.de](https://glad.dav1d.de/) 选择自己需要的版本下载并解压，得到如下目录。

<div style="display:flex;justify-content:center">
  <img src="/images/clion_setup/10.png"/>
</div>

<div style="display:flex;justify-content:center">
  <img src="/images/clion_setup/11.png"/>
</div>

将include/glad和include/KHR两个文件夹都复制到项目include目录下，src/glad.c复制到项目src目录下。

<div style="display:flex;justify-content:center">
  <img src="/images/clion_setup/12.png"/>
</div>

## 四、配置assimp

可以去GitHub仓库中拉取代码自己编译，或者下载预编译版本 https://kimkulling.itch.io/the-asset-importer-lib 官方提供的是一个exe安装包，下载后安装到任意目录。

<div style="display:flex;justify-content:center">
  <img src="/images/clion_setup/13.png"/>
</div>

<div style="display:flex;justify-content:center">
  <img src="/images/clion_setup/14.png"/>
</div>

来到安装目录，将include/assimp整个文件夹复制到项目下include目录中，lib/x64/assimp-vc143-mt.lib复制到项目下lib目录中，bin/x64/assimp-vc143-mt.dll复制到cmake-build-debug-vs2022目录中（你添加的vs配置文件对应的build目录中），之后就可以把安装的assimp卸载了。

<div style="display:flex;justify-content:center">
  <img src="/images/clion_setup/15.png"/>
</div>

## 五、其他配置

glm库：github克隆仓库后将glm文件夹复制到项目include目录下。

stb_image库：将stb_image.h放入include目录下。

此处不过多赘述。

<div style="display:flex;justify-content:center">
  <img src="/images/clion_setup/16.png"/>
</div>

## 六、配置cmakelists

在项目根目录下创建demos和headers两个目录，demos用于存放要运行的cpp源文件，headers用于存放自己写的头文件如camera.h，shader.h等。也可以不要headers目录直接把自己写的头文件也放在include下，但做好区分好一些。

之后在Cmakelists.txt写入以下内容：

```cmake
cmake_minimum_required(VERSION 3.30)
project(LearnOpengl) #括号内改为你自己的项目名

set(CMAKE_CXX_STANDARD 20)

include_directories(${PROJECT_SOURCE_DIR}/include ${PROJECT_SOURCE_DIR}/headers)
link_directories(${PROJECT_SOURCE_DIR}/lib)

file(GLOB files demos/*.cpp)
foreach (file ${files})
    string(REGEX REPLACE ".+/(.+)\\..*" "\\1" file_name ${file})
    add_executable(${file_name} src/glad.c ${file})
    target_link_libraries(${file_name} ${PROJECT_SOURCE_DIR}/lib/glfw3.lib)
    target_link_libraries(${file_name} ${PROJECT_SOURCE_DIR}/lib/assimp-vc143-mtd.lib)
endforeach ()
```

右键Cmakelists.txt选择重新加载Cmake项目，会自动为demos目录下每一个cpp文件生成一个可编译运行的配置，可以运行以下代码看看glfw和glad是否配置成功。

```cpp
#include <glad/glad.h>
#include <GLFW/glfw3.h>

#include <iostream>

void framebuffer_size_callback(GLFWwindow* window, int width, int height);
void processInput(GLFWwindow *window);

// settings
const unsigned int SCR_WIDTH = 800;
const unsigned int SCR_HEIGHT = 600;

const char *vertexShaderSource = "#version 330 core\n"
    "layout (location = 0) in vec3 aPos;\n"
    "void main()\n"
    "{\n"
    "   gl_Position = vec4(aPos.x, aPos.y, aPos.z, 1.0);\n"
    "}\0";
const char *fragmentShaderSource = "#version 330 core\n"
    "out vec4 FragColor;\n"
    "void main()\n"
    "{\n"
    "   FragColor = vec4(1.0f, 0.5f, 0.2f, 1.0f);\n"
    "}\n\0";

int main()
{
    // glfw: initialize and configure
    // ------------------------------
    glfwInit();
    glfwWindowHint(GLFW_CONTEXT_VERSION_MAJOR, 3);
    glfwWindowHint(GLFW_CONTEXT_VERSION_MINOR, 3);
    glfwWindowHint(GLFW_OPENGL_PROFILE, GLFW_OPENGL_CORE_PROFILE);

#ifdef __APPLE__
    glfwWindowHint(GLFW_OPENGL_FORWARD_COMPAT, GL_TRUE);
#endif

    // glfw window creation
    // --------------------
    GLFWwindow* window = glfwCreateWindow(SCR_WIDTH, SCR_HEIGHT, "LearnOpenGL", NULL, NULL);
    if (window == NULL)
    {
        std::cout << "Failed to create GLFW window" << std::endl;
        glfwTerminate();
        return -1;
    }
    glfwMakeContextCurrent(window);
    glfwSetFramebufferSizeCallback(window, framebuffer_size_callback);

    // glad: load all OpenGL function pointers
    // ---------------------------------------
    if (!gladLoadGLLoader((GLADloadproc)glfwGetProcAddress))
    {
        std::cout << "Failed to initialize GLAD" << std::endl;
        return -1;
    }


    // build and compile our shader program
    // ------------------------------------
    // vertex shader
    unsigned int vertexShader = glCreateShader(GL_VERTEX_SHADER);
    glShaderSource(vertexShader, 1, &vertexShaderSource, NULL);
    glCompileShader(vertexShader);
    // check for shader compile errors
    int success;
    char infoLog[512];
    glGetShaderiv(vertexShader, GL_COMPILE_STATUS, &success);
    if (!success)
    {
        glGetShaderInfoLog(vertexShader, 512, NULL, infoLog);
        std::cout << "ERROR::SHADER::VERTEX::COMPILATION_FAILED\n" << infoLog << std::endl;
    }
    // fragment shader
    unsigned int fragmentShader = glCreateShader(GL_FRAGMENT_SHADER);
    glShaderSource(fragmentShader, 1, &fragmentShaderSource, NULL);
    glCompileShader(fragmentShader);
    // check for shader compile errors
    glGetShaderiv(fragmentShader, GL_COMPILE_STATUS, &success);
    if (!success)
    {
        glGetShaderInfoLog(fragmentShader, 512, NULL, infoLog);
        std::cout << "ERROR::SHADER::FRAGMENT::COMPILATION_FAILED\n" << infoLog << std::endl;
    }
    // link shaders
    unsigned int shaderProgram = glCreateProgram();
    glAttachShader(shaderProgram, vertexShader);
    glAttachShader(shaderProgram, fragmentShader);
    glLinkProgram(shaderProgram);
    // check for linking errors
    glGetProgramiv(shaderProgram, GL_LINK_STATUS, &success);
    if (!success) {
        glGetProgramInfoLog(shaderProgram, 512, NULL, infoLog);
        std::cout << "ERROR::SHADER::PROGRAM::LINKING_FAILED\n" << infoLog << std::endl;
    }
    glDeleteShader(vertexShader);
    glDeleteShader(fragmentShader);

    // set up vertex data (and buffer(s)) and configure vertex attributes
    // ------------------------------------------------------------------
    float vertices[] = {
        -0.5f, -0.5f, 0.0f, // left
         0.5f, -0.5f, 0.0f, // right
         0.0f,  0.5f, 0.0f  // top
    };

    unsigned int VBO, VAO;
    glGenVertexArrays(1, &VAO);
    glGenBuffers(1, &VBO);
    // bind the Vertex Array Object first, then bind and set vertex buffer(s), and then configure vertex attributes(s).
    glBindVertexArray(VAO);

    glBindBuffer(GL_ARRAY_BUFFER, VBO);
    glBufferData(GL_ARRAY_BUFFER, sizeof(vertices), vertices, GL_STATIC_DRAW);

    glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 3 * sizeof(float), (void*)0);
    glEnableVertexAttribArray(0);

    // note that this is allowed, the call to glVertexAttribPointer registered VBO as the vertex attribute's bound vertex buffer object so afterwards we can safely unbind
    glBindBuffer(GL_ARRAY_BUFFER, 0);

    // You can unbind the VAO afterwards so other VAO calls won't accidentally modify this VAO, but this rarely happens. Modifying other
    // VAOs requires a call to glBindVertexArray anyways so we generally don't unbind VAOs (nor VBOs) when it's not directly necessary.
    glBindVertexArray(0);


    // uncomment this call to draw in wireframe polygons.
    //glPolygonMode(GL_FRONT_AND_BACK, GL_LINE);

    // render loop
    // -----------
    while (!glfwWindowShouldClose(window))
    {
        // input
        // -----
        processInput(window);

        // render
        // ------
        glClearColor(0.2f, 0.3f, 0.3f, 1.0f);
        glClear(GL_COLOR_BUFFER_BIT);

        // draw our first triangle
        glUseProgram(shaderProgram);
        glBindVertexArray(VAO); // seeing as we only have a single VAO there's no need to bind it every time, but we'll do so to keep things a bit more organized
        glDrawArrays(GL_TRIANGLES, 0, 3);
        // glBindVertexArray(0); // no need to unbind it every time

        // glfw: swap buffers and poll IO events (keys pressed/released, mouse moved etc.)
        // -------------------------------------------------------------------------------
        glfwSwapBuffers(window);
        glfwPollEvents();
    }

    // optional: de-allocate all resources once they've outlived their purpose:
    // ------------------------------------------------------------------------
    glDeleteVertexArrays(1, &VAO);
    glDeleteBuffers(1, &VBO);
    glDeleteProgram(shaderProgram);

    // glfw: terminate, clearing all previously allocated GLFW resources.
    // ------------------------------------------------------------------
    glfwTerminate();
    return 0;
}

// process all input: query GLFW whether relevant keys are pressed/released this frame and react accordingly
// ---------------------------------------------------------------------------------------------------------
void processInput(GLFWwindow *window)
{
    if (glfwGetKey(window, GLFW_KEY_ESCAPE) == GLFW_PRESS)
        glfwSetWindowShouldClose(window, true);
}

// glfw: whenever the window size changed (by OS or user resize) this callback function executes
// ---------------------------------------------------------------------------------------------
void framebuffer_size_callback(GLFWwindow* window, int width, int height)
{
    // make sure the viewport matches the new window dimensions; note that width and
    // height will be significantly larger than specified on retina displays.
    glViewport(0, 0, width, height);
}
```

<div style="display:flex;justify-content:center">
  <img src="/images/clion_setup/17.png"/>
</div>

之后可以自己随便找一个模型文件，通过以下代码看看assimp是否配置成功：

```cpp
#include <assimp/Importer.hpp>
#include <assimp/scene.h>
#include <assimp/postprocess.h>
#include <iostream>

int main() {
    Assimp::Importer importer;
    const aiScene* scene = importer.ReadFile("path_to_your_model.obj", aiProcess_Triangulate);
    if (!scene) {
        std::cerr << "Error: " << importer.GetErrorString() << std::endl;
        return -1;
    }
    std::cout << "Assimp OK, load model succeed." << std::endl;
    return 0;
}
```

学习到新的内容就可以直接在demos下创建一个新的cpp文件之后重新加载cmake项目，不用清除掉之前的内容，可以随时回顾自己学到的东西，自己写的头文件就放入headers目录下。

<div style="display:flex;justify-content:center">
  <img src="/images/clion_setup/18.png"/>
</div>

开始愉快地学习吧！

<div style="display:flex;justify-content:center">
  <img src="/images/clion_setup/19.png"/>
</div>

<div style="display:flex;justify-content:center">
  <img src="/images/clion_setup/20.png"/>
</div>

<div style="display:flex;justify-content:center">
  <img src="/images/clion_setup/21.png"/>
</div>
