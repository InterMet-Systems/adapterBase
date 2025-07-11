# 1 Version
0.1.0

# 2 Lexicon Format
A lexicon is a set of data representing the required initialization data, available commands, and source of the lexicon data that can be sent over an interface. In the context of DLLs or plugins, this means that The lexicon is provided by an adapter to describe what commands that adapter is capable of utilizing. Lexicons only contain information that is unique to the device!

Here's an example lexicon with tabs added to make it look nice:
/* all comments must fit within C++ style comment blocks, // is not allowed */
/*
name                    type            unit    cmds    desc
*/
"voltage"               :uint32e2	    :v      :q	    :"supply voltage";
"commandVoltage"        :uint32e-2	    :v      :q	    :"ESC commanded voltage";
"torque"                :uint32e1	    :nm     :qs	    :"current torque";
"maxTorque"             :uint32e1	    :nm     :qs	    :"maximum torque";
"speed"                 :int32e0		:rpm    :qs	    :"rotational velocity";
"maxSpeed"		        :int32e0		:rpm    :qs	    :"rotational velocity";
"angle"                 :int32e1	    :radian :qs	    :"angle relative to motor datum";
"controlMode"           :uint8e0		:null   :qs	    :"set mode, 1=torque, 2=speed, 3=position";
"state"                 :uint8e0		:null   :q	    :"state code for errors, refer to manual";
"escTemperature"        :int32e1	    :k      :q	    :"temperature of the ESC board";
"coilTemperature"       :int32e1	    :k      :q	    :"temperature of the motor coils";
"triggerMeasurement"    :null		    :null   :s	    :“request a measurement be immediately done”;
"example2"              :uint16e0[4]    :H      :qs     :"example";
"example3"              :<uint16e0>     :mF     :qs     :"example";
"example5"              :string         :ko     :qs     :"example";
"example6"              :blob           :mF     :qs     :"example";

Lexicons are ASCII encoded narrow strings consisting of colon and semicolon delimited data. Semicolons delimit command specifications (depicted as a row), and colons delimit tokens (depicted as columns). White space is ignored. All text is case sensative.

A command specification consists of 5 tokens:
name
type
unit
command
description

## 2.1 Name token
The name is a character string that is an alias for the command. All names within a lexicon must be unique. ASCII characters 32, 33, and 35-126 are allowed, omitting 34 (") to simplify parsing, for now. 

The maximum length of the string is 32 characters. This constraint is imposed to limit memory usage in systems that are pre-allocating memory.

## 2.2 Type token
This may not be necessary with Java, as types can be passed between plugins and the calling code seemlessly.

As shown in the example, the types are suitable for passing across barriers. That includes from a dynamic library and hardware transmission.

The type is both the argument type and return type. The presence of an argument or return is determined by declaring it a null type and the command token. At the moment, there is no plan to add separate argument and return types because the intention is to handle low level operations for the user.

Everything is an integer or null, null indicates no data is sent or received, just the base command (ie only the name is passed to the adapter)
uint32e2[4]
u = unsigned, omit for signed
32 = 32 bit, offer 8, 16, 32, and 64, but 64 will need adaptations for 32 bit systems if implemented on anything that compiles to it
e2 = multiply the contents by 10^2 to get the specified unit as per standard exponential notation
[4] = array of 4 elements, fixed length, always
<uint16e0> = vector of uint16e0, variable length allowed, allow at least 2^32 elements
string = alias for uint8e0 that signals to front ends to display the data as an ASCII character
blob = alias for uint8e0 that signals to front ends to display the data in an untyped form (probably hex)

No bool, for now.

### 2.2.1 Array Implementation
Arrays are difficult to pass from dynamic libaries. There is no way to pass a variable length container at the time of compilation to a dynamic library because it requires a signature that can be defined in both the host and dynamic library. Variable length containers are not fundamental types in C or C++ and are not supported in Java or Python. They are passed across the isolation barrier as a struct of void*, int representing a pointer to the first byte of a blob of length int. This technique requires that once data is shared it cannot have its length changed. This is acceptable for the application of reading and writing data to external devices.

So when I say uint32[4], what I'm actually saying is encode uint32[4] into little endian uint8_t[16], store uint8_t[0]* and 16 in a struct, then pass that struct to the caller. The caller reads the blob, translates endianness as needed, and casts the type based on the lexicon entry. What the pointer points to is irrelevant, so cast it to a void* to reduce the interface to one function and reference the lexicon to determine what it should be cast to (the array data will get passed with name, and the name is used as the lookup term in the lexicon). All common systems use the same format for pointers so long as the architecture is the same, which it always will be due to this taking place within an OS. Endianness is also likely to be the same, but the implementor must verify it. Because the arrays are a blob and the pointer is a void*, the DLL interface doesn't need any overloaded functions or whatnot to handle different kinds of data. This means that only one function is required to pass any data that can be represented by a lexicon to and from a plugin. For dynamic libraries, this is required. For java's dynamic compilation, it might not be but is to be determined.

The blob needs to always be formatted in little endian to ensure accurate interpretation by the caller. Endianness is easy to check and change in C++, python, and Java. Just ask GPT.

For arrays that are passed between a dynamic library and host, the originating side must ensure safety for the array's memory. Whatever side allocated that memory ABSOLUTELY MUST ensure that no other threads on that side access that memory while the other side needs it.

When the host passes data to the dynamic library, this is achieved by the host waiting for the dynamic library function to return before using the memory (requiring it to be a blocking function). If the dynamic library needs the data in the array after returning, it must copy it to a location that is guarenteed to be safe by the dynamic library. It must be that way because the DLL can't do thread safe communication to the host by any means other than a function return. Because this format blocks the host, functions of this sort should be designed to return promptly.

When the dynamic library returns data to the host, it must use the new keyword to make a copy of the data first (note that all dynamic libraries in Project Mycelium will be in C or C++). When the host is done with the point, it must call a release(void*) function to make the DLL release the memory. The void* must be passed so the dynamic library knows what pointer to release. There could be multiple! It is the responsibility of the host to call release or there will be a memory leak.

## 2.3 Unit token
The unit of the number, applicable for physical measurements. This is intended for automatic unit conversions, but won't be directly supported by individual adapters beyond ensuring that the unit is correct. Unit conversion will be supported by the adapter interface library.

A table of all common units needs to be added. A square conversion table needs to be made for each physical quantity.

String literals are not used because the expectation is to adhere to a standard set of units supported by the protocol. There may be reason to expand on that, but for now it can wait.

## 2.4 Command token
Determines what kind of command is available.
q = query, the type token represents a return value
s = set, the type token represents an argument

## 2.5 Description token
The description is a short explanation of what the name represents or the command does. This is mainly indended for display in an system automatically detecting adaptors. ASCII characters 32, 33, and 35-126 are allowed, omitting 34 (") to simplify parsing, for now. 

The maximum length of the string is 64 characters. This constraint is imposed to limit memory usage in systems that are pre-allocating memory.







Change readme to: make the com libraries with C/C++. Make the adapters and front end out of Java. This way only the platform specific code must be changed with the platform. The lexicon is still needed for the adapter<->front end coms because the adapters still require a simplified common interface. Types may or may not be necessary. Consider adding units to the lexicon and defining standard unit conversions.
