This solution contains a simple collaborative drawing UWP app built on top of Ambrosia, along with
a .NET Framework version of the app. Both the UWP version and the .NET Framework version use the
same type of Immortal to communicate with each other.

Known issues with UWP
=====================

The first two issues have to do with properly setting up the project structure of a UWP app using
Ambrosia. The solution in this directory provides an example of how to address both issues.

* UWP projects have a very different csproj file layout than .NET Framework or .NET Core projects,
so I don't think it's possible to have a csproj file that contains both .NET Framework/.NET Core
and UWP targets. UWP projects also complain when you try to add a reference to a project with
multiple target frameworks, even if one of those frameworks is compatible with UWP. As a result, I
had to make UWP clone projects of the Ambrosia and AmbrosiaLibCS projects with links to the
original source files. I also had to make a UWP clone project for any project in my solution
referenced by both UWP and non-UWP apps that directly references Ambrosia or AmbrosiaLibCS
(e.g., the project with my Immortal implementation), since the versions of these projects used by a
UWP app need to reference the UWP versions of Ambrosia and AmbrosiaLibCS.

* (Related to the previous point) Because the CodeGen program generates a project that targets
.NET Framework and .NET Core, we also have to have a UWP clone of that generated project.
Currently, the generated source files are copied into the UWP clone project by this solution's
Generate-Assemblies.ps1 script, but in the future, we could think about adding support into the
CodeGen program for generating a UWP version of the generated project (or adding a UWP target to
the generated project if multi-targeting with a UWP target is possible).

* CRA normally uses dynamic assembly loading to instantiate vertices, but UWP does not support
dynamic assembly loading. As a result, to run an AmbrosiaRuntime instance as a vertex in a CRA
Worker, you'll need to use the "vertex sideloading" mechanism that we developed as a workaround.
When sideloading a vertex, you instantiate the vertex manually, call `CRAWorker.SideloadVertex()`
with the vertex instance and its name as arguments, and only then call `CRAWorker.Start()`. The
GraphicalAppUWP project in this directory provides an example of launching a CRA Worker with a
sideloaded AmbrosiaRuntime vertex.

* By default, UWP apps are sandboxed so that they can only read and write files in their individual
app folders. Ambrosia is currently built so that you choose a log directory when you run
`Ambrosia.exe` to define an Ambrosia service, and the Ambrosia runtime assumes that this log
directory will be accessible on any machine by any app that deploys that service. As a result,
we've been using log directory paths like `C:\logs`. In order to allow GraphicalAppUWP to access
arbitrary log directories, I added the restricted capability `broadFileSystemAccess` to its
manifest. For UWP apps deployed in production or submitted to the Windows Store, the ideal solution
is to ensure that Ambrosia uses a log directory inside of the app's app folder. Implementing that
solution may require making changes to (or adding special cases to) Ambrosia's process for defining
a service's log directory.

* When a UWP app starts both a CRAWorker and its corresponding Immortal in separate threads, it takes
a long time for the connections between the two to be established due to apparent TCP timeout
issues on localhost in UWP. Specifically, if thread A opens a socket and tries to connect to port
X, and thread B soon after opens a socket that binds to port X and listens, thread A's initial
connection attempt will time out, but it will take a long time to do so (~20 seconds). This
behavior has nothing to do with Ambrosia and is reproducible in a barebones UWP app. This
particular connection pattern occurs in both directions of setting up the connections between
the CRAWorker and the Immortal, causing startup to take around 40 seconds to finish.

Things remaining to do
======================
* Decide whether to move the UWP clone projects of Ambrosia and AmbrosiaLibCS somewhere
more centralized so that other developers can use them, or leave them in the UWPExample directory.

Projects in this solution
=========================

GraphicalAppUWP
---------------
The UWP collaborative drawing app.

GraphicalApp
------------
A Windows Forms version of the collaborative drawing app.

GraphicalImmortal
---------------
Contains the implementation of the Ambrosia Immortal used by the collaborative drawing apps.

GraphicalImmortalAPI
------------------
Contains the IDL interface of the GraphicalImmortal. This project is not referenced by any other
project and is used only for codegen.

AmbrosiaUWP and AmbrosiaLibCSUWP
---------------------------------------
Clones of the Ambrosia and AmbrosiaLibCS projects (with links to the original source files)
configured to compile as UWP class libraries. UWP clones of these projects were required due to the
issues mentioned above.

GraphicalImmortalUWP
------------------
A clone of the GraphicalImmortal project configured to compile as a UWP class library. This project
has to be a UWP class library rather than a .NET Standard class library because it needs to
reference the UWP versions of Ambrosia and AmbrosiaLibCS.

GraphicalImmortalAPIGeneratedUWP
------------------------------
A project containing the source files generated by the codegen tool based on the IDL interface in
GraphicalImmortalAPI. This project is the UWP analogue to the .NET Framework/.NET Core project
produced by the codegen tool. The PowerShell script "Generate-Assemblies.ps1" takes care of copying
the source files produced by the codegen tool into this project.