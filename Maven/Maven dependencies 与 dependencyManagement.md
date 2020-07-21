### Maven dependencies 与 dependencyManagement

**dependencies **即使在子项目中不声明该依赖项，子项目仍然会从父项目中继承该依赖项（全部继承）。

**dependencyManagement** 里只是声明依赖，并不实现引入，因此子项目需要显示的声明需要用的依赖。如果不在子项目中声明依赖，则不会从父项目中继承下来；只有在子项目中写了该依赖项，并且没有指定具体版本，才会从父项目中继承该项，并且 version 和 scope 都读取自父 pom。另外如果子项目中指定了版本号，那么会使用子项目中指定的版本。
