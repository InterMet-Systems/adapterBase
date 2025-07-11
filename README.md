# 1: Context: Project Mycelium
The adapterBase repository is part of Project Mycelium. If you want to understand why adapterBase exists and is how it is, start here.

## 1.1: Why Project Mycelium?
This repository is center of the Mycelium Project.

The Mycelium Project is an effort to create a flexible backend architecture for data acquisition and processing software. Example applications include automated quality assurance, sensor data acquisition programs, and device control software. Within that domain of applications, it is common for multiple roles to be mashed together in a single repository to make a highly custom application. This has numerous downsides. Code is difficult to copy and adapt to new applications, resulting in repeated work. Updates to even basic routines are difficult to propogate to other programs. Dissimilar areas of expertise are combined, resulting in complex code that is simultaneously addressing human interfaces, device communication, and data processing all at once. Such code bases are difficult and expensive to maintain. The goal of the Mycelium Project (or Project Mycelium) is to create an architecture that is not those things.

To be specific, Project Mycelium's goals are to:
1. Reduce the work load on programmers by using automation to propogate code changes between projects.
2. Increase the commonality between projects to reduce the amount of code that must be maintained to fulfill a function within an organization.
3. Isolate logic of data acquisition from data processing, allowing application logic to be expressed with less knowledge of how hardware operates.

## 1.2 Original Author's Intentions
The concept of project mycelium requires would be beneficial to a wide variety of devices. For the original authors, Windows platforms targeting x86 are the most valuable and are therefore the focus of early development. However, linux on x86 and ARM 32 and 64 bit are a potentially valuable expansion due to the potential applications on industrial computers and single board computers. A variety of other factors were also considered.

It was determined the best approach was to use Java for the front end and adapters. This provides strong multiplatform support for x86, ARM, multiple platforms (Windows, linux, Mac), and the technical requirements for a high-performance implementation. However, Java's support for low level OS operations, especially those seen on single board computers that expose functions not typically available on a PC, is to use JNI or JNA with C implementatons. Instead, dynamic libraries will be written in C or C++ to support edgecases that would require JNI.

As of 7/11/25, Java 17 is the newest Java version that does everything described. https://www.azul.com/downloads/?version=java-17-lts&package=jdk#zulu

## 1.3 Architecture of Project Mycelium
The general structure of the Mycelium Project is:
- (C++, C) Communication libraries (for ex TCP, UART, win API) are the lowest level custom code for this project. These should be the lowest level of git repositories and should have all the source code necessary to function in the repository. These libraries provide moderately low level interfaces that hide OS level API calls, most protocol specific logic, and synchronicity logic. Public interfaces excepting initialization should be asynchronous.
- (Java) Adapters have a base class, adapterBase (this repository), that use the communication libraries via git submodules. An adapter is a plugin based wrapper for device specific logic, such as framing and encoding of messages. Adapters are loaded as plugins by using Java's dynamic compilation feature.
- (Java) Adapters include adapterBase as a git submodule and inherit from adapterBase. Device specific logic is only defined in an adapter.
- (Java) Applications include adapterBase as a git submodule in order to access the adapter interface library. They then use it to open and read dynamic libraries using that library.

Note that only one level of inheritance is possible in Java, so requiring implementors to inherit from adapterBase means that no repositories can be made that use inheritance to build on an adapter. That is an acceptable limitation to the original authors.

What this architecture achieves:
- A high degree of automation in version control by heavily relying on git to track different functional sections of code. Contributes to goal #1.
- High commonality, with dynamic libraries and the plugin interface shared across all permutations of device specific code. Contributes to goal #2.
- All back end functions are separated from human interfaces and application logic. Contributes to goal #3.
- OS functions that are outside of Java's normal domain are separated from the device specific logic. Contributes to goal #3.

### 1.3.1: Why not a human interface?
No human interface standard is provided. This is intentional. The degrees of variation in human interfaces used in a project range from none to extremely complex, from highly curated to extremely basic. Furthermore, customized GUI library would more than double the scope of Project Mycelium. There is also virtually no commonality in the code for a GUI and the code for device communication.

# 2: Components of this repository
adapterBase source code
adapter example
adapter interface library
adapter test program using the adapter example and adapter interface library
lexicon specification
