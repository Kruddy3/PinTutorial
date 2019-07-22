# Installing pin on windows and building your first pin tool
This guide is set up in a way to help beginners successfully build pin tools and run them on a windows machine. The first section will be installing and setup. 
We will use Visual Studios 2015 to build the tools and Cygwin to actually run the tool.

## Part 1
### Installing Pin
- [Pin Download link](https://software.intel.com/en-us/articles/pin-a-binary-instrumentation-tool-downloads)

- [Docs for more information](https://software.intel.com/sites/landingpage/pintool/docs/97619/Pin/html/index.html)

	- extract and rename the folder to pin
	- move pin folder to C:/

### Installing VS2015
- Install Visual Studio 2015 (2017 can work for 3.10 according to docs)
- [Join Microsoft dev essentials so that you can get an older version of MSVS](https://my.visualstudio.com/subscriptions)

![Image](img/1.png)
 
- [Then you can download it from here (be careful of x64 vs x86)](https://my.visualstudio.com/Downloads?q=Visual%20Studio%202015%20with%20Update%203)
	- Make sure to install the c/c++ workspaces 

### Building The Tools
- Open C:\pin\source\tools\MyPinTool\MyPinTool.vcxproj in Visual Studio 

![Image](img/2.png)

- expand the Source Files section.

![Image](img/3.png)

- Edit the MyPinTool.cpp file and replace its content with the source code of your tool but to test you can use inscount0.cpp found at C:\pin\source\tools\ManualExamples\inscount0.cpp
- Ensure you set Release as Solution Configuration option:

![Image](img/4.png)

- Right click on MyPinTool and select Properties
- Go to VC++ directories > Include Directories and add the following paths:
	C:\pin\source\include\pin;
	C:\pin\source\include\pin\gen;
	
![Image](img/5.png)

- Add the following to Configuration Properties -> C/C++ -> Additional Include Directories
	..\..\..\extras\xed-ia32\include\xed
	
![Image](img/6.png)

- Add this to Configuration Properties -> Linker -> Input -> Additional Dependencies
	crtbeginS.obj
	
![Image](img/7.png)

- Set Configuration Properties -> Linker -> Advanced -> Image Has Safe Exception Handlers to
	No (/SAFESEH:NO)
	
![Image](img/8.png)

You should be able to build the tools now with no errors (Ctrl+Shift+B or right click MyPinTool->Build)

### Transferring built tool to the School Computer
- Connect to the school vpn (anyc.vpn.gatech.edu) using Cisco AnyConnect Mobility Client
- Download WINSCP to transfer your built tool
- WINSCP to the prism server 
	- Hostname: scp.prism.gatech.edu

![Image](img/9.png)	 

- Navigate to: 
```
	/nethome/{YOUR_USERNAME}/ECE/Desktop
```
Upload the zipped folder of MyPinTools (C:\pin\source\tools\MyPinTool)

### Transferring Built tool to Malware VM using RDP
- To connect to the computer surrounding the VM you need to use xfreerdp 
- The command will look something like:
```
	xfreerdp /u:GTUSERNAME /d:AD /v:ece-4894-01.ece.gatech.edu 
```
	- If you are getting a certificate warning you may need to add /cert-ignore
	- If you would like to change the dimensions add /w:1800 /h:900

#### Transferting using shared folders

- If you are transfering files between the malware VM and the surrounding VM using shared folders it is very easy. You can actually share the folder where that you are building your pin tool in to make it easy.

#### Transfering using COM ports

- At this point the tool should have been built on your computer then compressed and transferred to the schools servers. We will now transfer the tool using a the COM1 port.
- Move the file from where you uploaded it on the desktop to C:\vm_setup_bds
![Image](img/10.png)
- Open PowerShell on the 4894 computer you have connected to:
- Use instruction (make sure the malware VM is open): 
```
	Copy-vmfile ece-4894-vm c:\ -SourcePath C:\vm_setup_bds\FILE -createfullpath -filesource Host
```
![Image](img/11.png)
- Unpack it and place it in the same place on pin (C:\pin\source\tools\MyPinTool on the vm)
- Open Cygwin on desktop 
- Navigate to c:/pin

```
	cd /cygdrive/c/pin
```

- Use this command to test if it works

```
c:/pin> ./pin.exe -t ./source/tools/MyPinTool/Release/MyPinTool.dll -- ./../Windows/System32/calc.exe
	this command has 3 parts
	PINEXE -t THE_TOOL_DLL_BEING_USED -- THE_PROGRAM_BEING_TESTED
```
- output file should be in c:/pin and the console should also output the result

![Image](img/12.png)

Transfer the files out using this as reference: [Link](https://charbelnemnom.com/2016/04/how-to-copy-files-between-the-guest-and-the-host-in-hyperv-with-powershell-direct/)
	- The problem with transfering files out using com ports is that it requires permissions to do so

## Part 2
### Making the pin tool
First a warning. The program I will be working on is malware which is likely what people using this tutorial will also be working on. I have fully reverse engineered the program I will be working with, so I am confident that there will be no negative effects for the pin tool I will be writing.
The main command we will be relying on is:
PIN_AddSyscallEntryFunction
I will be finishing up the pin tool I am working on next week when my professor looks it over and someone who sets up the servers finishes one last part. I can not confidently complete the portions needed to continue this section of the malware
Take some time to read over the API references [here](https://software.intel.com/sites/landingpage/pintool/docs/97971/Pin/html/group__API__REF.html) to be familiar with the terms that will be used.

### Learning the basics
A pin tool has 2 major sections: 
	- Instrumentation routine
	- Analysis routine

The instrumentation routine gets called with every instruction and it is where we decide what to do with the instruction given. This routine can place our own functions where we want in the program we are analyzing. 
The analysis routine defines what to do when the instrumentation is activated. This is typically where the actual goal of our tool is completed.

I will be walking through the example program malloctrace which can be found at pin/source/tools/ManualExamples/malloctrace.cpp
### Breaking down malloctrace
This tool walks through a program and shows the inputs when malloc is called and what it returns.
``` malloctrace.cpp
#include "pin.H"
#include <iostream>
#include <fstream>

#if defined(TARGET_MAC)
#define MALLOC "_malloc"
#define FREE "_free"
#else
#define MALLOC "malloc"
#define FREE "free"
#endif

//This is where we will be storing the output of our tool
std::ofstream TraceFile; 

/* Knobs automate the parsing and management of command line switches. A command line contains switches for Pin, the tool, and the application. The knobs parsing code understands how to separate them. */
KNOB<string> KnobOutputFile(KNOB_MODE_WRITEONCE, "pintool",
    "o", "malloctrace.out", "specify trace file name");
/* ===================================================================== */
/* Analysis routines                                                     */
/* ===================================================================== */

//we are getting the inputs from our instrumentation routines and printing them to the file
VOID Arg1Before(CHAR * name, ADDRINT size)
{
    TraceFile << name << "(" << size << ")" << endl;
}

VOID MallocAfter(ADDRINT ret)
{
    TraceFile << "  returns " << ret << endl;
}


/* ===================================================================== */
/* Instrumentation routines                                              */
/* ===================================================================== */
   
VOID Image(IMG img, VOID *v)
{
    //we search through the img for a routine named malloc or _malloc
    RTN mallocRtn = RTN_FindByName(img, MALLOC);

//if we found a routine in the image with that name continue
    if (RTN_Valid(mallocRtn))
    {
        RTN_Open(mallocRtn);

        //we are inserting our own call to our own functions here but lets break it down more
        RTN_InsertCall(
//the RTN that is getting passed to the analysis routine 
	mallocRtn, 
//we are inserting our call BEFORE the routine is ran
IPOINT_BEFORE, 
//Arg1Before is the function we will be running when this gets triggered
(AFUNPTR)Arg1Before,
//IARG_ADDRINT is the datatype we are passing in to Arg1Before
IARG_ADDRINT, 
//Malloc is the value being passed to the function
MALLOC,
//We are grabbing the 1st (0 position) of the function parameters we found and passing it to Arg1Before
//for malloc it has 1 parameter which is size
IARG_FUNCARG_ENTRYPOINT_VALUE, 0,
//all argument lists must end with this
IARG_END);

        RTN_InsertCall(
mallocRtn, 
//we are setting a call immediately AFTER the routine has finished
IPOINT_AFTER, 
(AFUNPTR)MallocAfter,
//we are getting the return value from the routine and passing it to our function
IARG_FUNCRET_EXITPOINT_VALUE, 
IARG_END);

        RTN_Close(mallocRtn);
    }

    // Find the free() function.
	//same as with malloc except to insert point after
    RTN freeRtn = RTN_FindByName(img, FREE);
    if (RTN_Valid(freeRtn))
    {
        RTN_Open(freeRtn);
        // Instrument free() to print the input argument value.
        RTN_InsertCall(freeRtn, IPOINT_BEFORE, (AFUNPTR)Arg1Before,
                       IARG_ADDRINT, FREE,
                       IARG_FUNCARG_ENTRYPOINT_VALUE, 0,
                       IARG_END);
        RTN_Close(freeRtn);
    }
}

//This function is called when the program finishes
VOID Fini(INT32 code, VOID *v)
{
    TraceFile.close();
}

/* ===================================================================== */
/* Print Help Message                                                    */
/* ===================================================================== */
INT32 Usage()
{
    cerr << "This tool produces a trace of calls to malloc." << endl;
    cerr << endl << KNOB_BASE::StringKnobSummary() << endl;
    return -1;
}

/* ===================================================================== */
/* Main                                                                  */
/* ===================================================================== */

int main(int argc, char *argv[])
{
/*Initialize symbol table code. Pin does not read symbols unless this is called.Must call before PIN_StartProgram

Symbol Table is an important data structure created and maintained by the compiler in order to keep track of semantics of variable i.e. it stores information about scope and binding information about names, information about instances of various entities such as variable and function names, classes, objects, etc. */
    PIN_InitSymbols();
    if( PIN_Init(argc,argv) )
    {
        return Usage();
    }
    
    // Write to a file since cout and cerr maybe closed by the application
    TraceFile.open(KnobOutputFile.Value().c_str());
    TraceFile << hex;
    TraceFile.setf(ios::showbase);
    
    // Use this to register a call back to catch the loading of an image.
//This is IMG_AddInstrumentFunction meaning an image is passed to our instrumentation function
	//Image is the function that will be called when this gets triggered
    IMG_AddInstrumentFunction(Image, 0);
//when the program finishes Fini will be called
    PIN_AddFiniFunction(Fini, 0);

    // Never returns
    PIN_StartProgram();
    
    return 0;
}
```


