# WebGPU Execution Environment

Note: Many items are marked "TBD".  In most cases those items will be resolved once the exact feature set of WebGPU has been determined.

## General WebGPU Environment

[TBD: this may belong in a different specification, if it exists, about WebGPU]

### Introduction

WebGPU executes graphics and compute pipelines.
A graphics pipeline processes data through a programmable vertex shader, and the resulting fragments through a programmable fragment shader.
A compute pipeline processes data via a compute shader.

A WebGPU pipeline defines:

* Shader modules for each programmable stage (vertex, fragment, and compute)
  * Shader modules declare their usage of resources as "bindings", "vertex inputs" or "fragment outputs".
    This is the shader module's implicit resource interface.
* Specialization constants for each shader module
* A "resource interface" that must be a superset of each of the shader module implicit interfaces.
  Each resource in a resource interface can be either read-only or read-write.
* Various "fixed function" configuration options.

WebGPU shaders are expressed in SPIR-V.  SPIR-V is usable in several execution environments, and so its core definition describes a superset of functionality across all those environments.
The "WebGPU Execution Environment Specification for SPIR-V" section describes restrictions on SPIR-V for use with WebGPU.

### Security Model

A WebGPU application (an "App") consists of code to invoke WebGPU API entry points ("host code"), and WebGPU shaders.
A WebGPU implementation does not trust either host code or shaders.

Stages of execution:

1. A WebGPU implementation loads App code, and begins running the App host code in an untrusted manner.
1. The App host code invokes WebGPU APIs to create resources, load shaders, and create a pipeline configuration.
1. The App invokes a WebGPU API to run a pipeline.

During these steps the WebGPU implementation will have:

1. Validated the pipeline configuration and all associated objects.
1. Determined the set of data resources that should be accessible to the shaders, and in what access modes (e.g. read-only, read-write).
1. Determined the computing resources to be used to execute the pipeline.
1. Created a pipeline ExecutionContext consisting of the computing and data resources that are valid to be used by the pipeline, and in particular by the shaders in the pipeline. 
   Each data resource is associated with valid access modes: read-only resources may only be read; only writable resources may be written.
1. Executes the pipeline, subject to the security property that follows.

A WebGPU implementation is _secure_ if, for any application _App_ running on the implementation, accesses explicitly granted to App are the only means by which:

*   _App_ may sense state outside its ExecutionContext
*   _App_ may cause mutation of state outside its ExecutionContext
*   _App_ may cause or sense I/O outside its ExecutionContext

An access is considered explicitly granted only if:

*   It is a read access to a resource attached for reading to the ExecutionContext
*   It is a write access to a resource attached for writing to the ExecutionContext

Typically a WebGPU implementations can make sure that the _App_ can only access resources provided in the ExecutionContext by validating the code in the shader module.
However this only guarantees that the correct resources are accessed, not that they are accessed in bounds.
This is an issue because resource accesses are often lowered to pointer arithmetic so out of bounds accesses could result in accesses outside the ExecutionContext.
For this reason out of bound accesses to resources must be prevented.

Validating accesses are in bounds at shader module or pipeline creation time isn't possible because it is equivalent to the halting problem.
Instead WebGPU implementation should make sure out of bounds accesses don't cause accesses outside of the ExecutionContext.
This will be typically done by instrumenting the generated code, and because of the performance sensitive nature of it, implementation will have a choice in the behavior of each out of bounds access.
Similarly to OpenGL's `GL_KHR_robust_buffer` access extension out of bounds accesses produces most produce the following behaviors:

* Be discarded for writes
* Access any location within the resource for reads and writes
* Return zero values for reads or (0, 0, 0, X) with X being 0, 1, -1, or extrema for integers, or -0.0, +0.0, -1.0, +1.0 for floating point values
* Atomics can return undefined values.

The robust resource access behavior of OpenGL is extended to allow for trapping behavior as well:

* Cause a trap that immediately stops execution of the shader, and fills all of the shader stage's output with zeroes.

Additionnally functionality exists to create pointers to values inside resources.
Creating a pointer for a resource that points to outside the resource could also result in the following behavior:

* Cause a trap.
* Defer the out of bounds behavior to when the pointer is dereferenced.
* Create a pointer to any location within the resource. (this is equivalent to deferring and "access any location" for resource accesses).

(TODO are there pointers to unsized arrays? Could talk about the "sized part of the type if that's the case")

## WebGPU Execution Environment Specification for SPIR-V

### Introduction

WebGPU is a client API for SPIR-V. Valid s<span style="color:#222222;">haders for WebGPU are defined by this document in addition to the [Khronos SPIR-V Specification, version 1.3, revision 5](https://www.khronos.org/registry/spir-v/specs/unified1/SPIRV.html)<span style="color:#222222;">.<span style="color:#222222;">  [TBD: update to latest available specification] See the "Version" subsection below for allowed version numbers.</span></span></span>

The following sections visit the parts of a SPIR-V module, in order, covering each section, and state additional requirements for using shaders in WebGPU. These are in addition to all requirements already present in the SPIR-V specification itself.

### Undefined Behaviour

The SPIR-V specification states that certain actions by a SPIR-V shader result in _undefined_ _behaviour_.  A WebGPU implementation is required to be secure.  Therefore in the context of WebGPU, a shader which triggers SPIR-V's undefined behaviour may continue to execute in an arbitrary manner except that it may not sense or mutate any state outside its ExecutionContext, and it may not cause or sense I/O outside its ExecutionContext.

NOTE: One of the possible behaviours of an ill-behaved application is non-termination.  The WebGPU specification should say what happens to a runaway application. TBD

### Out of bounds

Operations that are considered as out of bounds in SPIR-V (that's valid for WebGPU) are the following:

* `OpLoad`, `OpRead`, `OpMemoryCopy` when the pointer is out of bounds.
* `OpAccessChain` when any of the indices for array accesses are larger (or equal) to the array's size or when the pointer is out of bounds.
  Note that the size of arrays might known at runtime for `OpTypeRuntimeArray`.
* `OpInBoundsAccessChain` under the same conditions as `OpAccessChain`, effectively making it an exact equivalent of `OpAccessChain`.
* All operations starting with `OpAtomic` when the pointer is out of bounds.
* `OpImageFetch`, `OpImageRead`, `OpImageWrite` when coordinates aren't in bounds of the resource.
* `OpImageTexelPointer` when the coordinates aren't in bounds of the resource.

Other operations can result in out of bounds even if they aren't on resources. The same out of bounds rule apply to them as well:

* `OpVectorIndexDynamic` and `OpVectorInsertDynamic` when the index is larger (or equal to) the vector size.
* `OpBitFieldInsert`, `OpBitFieldSExtract` and `OpBitFieldUExtract` when the range of bits isn't fully contained in the variable.

### Version

The version number (second 32-bit word) of a SPIR-V module must be one of the following:



*   0x00010300 (version 1.3) [TBD: how far back do we need to go to get broad-enough support?]
*   … TBD: Subsequent version numbers with additional constraints that the functionality in the module can be rewritten by the browser to downgrade it to version 1.3.
    *   restriction 1 TBD
    *   restriction 2 TBD
    *   ...
*   … 


### Capabilities

All capabilities declared by **OpCapability** instructions (located after the header in a SPIR-V module) must be one of the following:



*   **Matrix**
*   **Shader**
*   **Sampled1D**
*   **Image1D**
*   **DerivativeControl**
*   **ImageQuery**
*   **SampledBuffer**? (needs investigation)
*   **ImageBuffer**? (needs investigation)
*   **VulkanMemoryModelKHR**
*   [Notably missing: **Linkage**, **Kernel**, …]

Further, if a capability is used only for non-enabled optional functionality, it is invalid to use it.


### Extensions

All extensions declared by **OpExtension** instructions must be one of the following:



*   SPV_KHR_vulkan_memory_model
*   [No other extensions allowed for now]


### Extended Instruction Sets

All extended instruction sets declared by **OpExtInstImport** must be one of the following:



*   "GLSL.std.450"


### SPIR-V Memory Model

The addressing declared by **OpMemoryModel** must be **Logical**. The memory model declared must be **VulkanKHR**.


### Entry Points

There must be at least one **OpEntryPoint** in a module, and each one present must satisfy:



*   The **OpFunction** declared as the entry point must have no **OpFunctionParameter** and  its _Return Type_ must be **OpTypeVoid**.
*   The complete call graph rooted by the entry point must
    *   be present in the module (the **Linkage** capability is not valid, so there are no partially linked modules), and
    *   not contain any cycles (no recursion, even statically).
*   The _Name_ operand must be a string of length [TBD, TBD] and cannot be used by any other **OpEntryPoint** in the same module.  All entry-point names in a module must be unique module these rules. TBD do we want case sensitive?

The execution model declared by each **OpEntryPoint** must be one of the following:



*   **Vertex**
*   **Fragment**
*   **GLCompute**


### Execution Modes

All execution modes declared by **OpExecutionMode** must be one of the following:



*   **OriginUpperLeft**
*   **DepthReplacing**
*   **DepthGreater**
*   **DepthLess**
*   **DepthUnchanged**
*   **LocalSize**
*   **LocalSizeHint**
*   … TBD


### Debug Instructions

All debug instructions must be stripped.


### Annotations

OpDecorationGroup, OpGroupDecorate, OpGroupMemberDecorate are not allowed.

Only the following decorations are valid (TBD):



*   TBD: **RelaxedPrecision**
*   **SpecId**
*   **Block**
*   **RowMajor**
*   **ColumnMajor**
*   **ArrayStride** (also see "Layouts" below)
*   **MatrixStride** (also see "Layouts" below)
*   **Builtin** (also see "Built-In Variables" below)
*   **NoPerspective**, only in the **Input** storage class and the **Fragment** execution model
*   **Flat**, only in the **Input** storage class and the **Fragment** execution model
*   **Centroid**
*   TBD: **Sample**
*   TBD: **Invariant**
*   TBD: **Restrict**
*   TBD: **Aliased**
*   TBD: **Volatile**
*   TBD: **Coherent**
*   **NonWritable**
*   **NonReadable**
*   TBD: **Uniform**
*   **Location**, required for all **Input** and **Output**, see "Interface" section below
*   **Component**
*   **Index**
*   **Binding**
*   **DescriptorSet**
*   **Offset** (also see "Layouts" below)
*   TBD: **FPRoundingMode**
*   **NoContraction**
*   **NoSignedWrap**
*   **NoUnsignedWrap**


#### Built-In Variables

When decorating with the **Builtin** decoration, only the following are valid, and these are further restricted as indicated by the table. (Table is TBD)


<table>
  <tr>
   <td><strong>Builtin</strong>
   </td>
   <td>Allowed types
   </td>
   <td>Allowed execution models for the <strong>Input</strong> storage class
   </td>
   <td>Allowed execution models for the <strong>Output </strong>storage class
   </td>
  </tr>
  <tr>
   <td><strong>Position</strong>
   </td>
   <td>Vector of 4 32-bit floats
   </td>
   <td>
   </td>
   <td><strong>Vertex</strong>
   </td>
  </tr>
  <tr>
   <td><strong>VertexIndex</strong>
   </td>
   <td>Scalar 32-bit integer
   </td>
   <td><strong>Vertex</strong>
   </td>
   <td>
   </td>
  </tr>
  <tr>
   <td><strong>InstanceIndex</strong>
   </td>
   <td>Scalar 32-bit integer
   </td>
   <td><strong>Vertex</strong>
   </td>
   <td>
   </td>
  </tr>
  <tr>
   <td><strong>FrontFacing</strong>
   </td>
   <td>Boolean
   </td>
   <td><strong>Fragment</strong>
   </td>
   <td>
   </td>
  </tr>
  <tr>
   <td><strong>FragCoord</strong>
   </td>
   <td>Vector of 4 32-bit floats
   </td>
   <td><strong>Fragment</strong>
   </td>
   <td>
   </td>
  </tr>
  <tr>
   <td><strong>FragDepth</strong>
   </td>
   <td>32-bit float
   </td>
   <td>
   </td>
   <td><strong>Fragment</strong>
   </td>
  </tr>
  <tr>
   <td><strong>NumWorkgroups</strong>
   </td>
   <td>Vector of 3 32-bit ints
   </td>
   <td><strong>GLCompute</strong>
   </td>
   <td>
   </td>
  </tr>
  <tr>
   <td><strong>WorkgroupSize</strong>
   </td>
   <td>Vector of 3 32-bit ints
   </td>
   <td><strong>GLCompute</strong>
   </td>
   <td>
   </td>
  </tr>
  <tr>
   <td><strong>LocalInvocationId</strong>
   </td>
   <td>Vector of 3 32-bit ints
   </td>
   <td><strong>GLCompute</strong>
   </td>
   <td>
   </td>
  </tr>
  <tr>
   <td><strong>GlobalInvocationId</strong>
   </td>
   <td>Vector of 3 32-bit ints
   </td>
   <td><strong>GLCompute</strong>
   </td>
   <td>
   </td>
  </tr>
  <tr>
   <td><strong>LocalInvocationIndex</strong>
   </td>
   <td>32-bit int
   </td>
   <td><strong>GLCompute</strong>
   </td>
   <td>
   </td>
  </tr>
  <tr>
   <td>TBD
   </td>
   <td>
   </td>
   <td>
   </td>
   <td>
   </td>
  </tr>
</table>


Full semantic descriptions of what these mean, how the behave, when they have what values, etc. is part of the WebGPU specification proper, not this specification.


### Types

Scalar floating-point types (**OpTypeFloat**) must have one of the following widths:



*   32 bits
*   TBD

Scalar integer types (**OpTypeInteger**) must have one of the following widths:



*   32 bits
*   TBD

Vector types must have one of the following number of components:



*   2
*   3
*   4

Matrix types must have one of the following number of vectors:



*   2
*   3
*   4

Storage Class must be one of the following:



*   **UniformConstant**
*   **Uniform**
*   **StorageBuffer** (no **BufferBlock**)
*   **Input**
*   **Output**
*   **Image**
*   **Workgroup**
*   **Private**
*   **Function**
*   TBD: **PushConstant?**

**OpTypeRunTimeArray** is only for the last member of a block in the **StorageBuffer** storage class.


### Functions, Declarations, and Execution


#### Functions



*   If function A calls function B, then B precedes A in the module.
*   OpTypeFunction may not have OpTypeVoid as a parameter type.


#### CFGs

Each block **_B_** in a function must satisfy one of the following rules:



*   **_B_** is the entry block (the first block listed in the function body), or there is a control flow path from the entry block to **_B_**.  (We say **_B_** is _reachable._)
*   There is no control flow path from the entry block to **_B_**, but:
    *   **_B_** is named as the merge-block for a merge instruction in a reachable block.  Additionally there are no branches to **_B_**, and **_B_** contains only an OpLabel and an OpUnreachable instruction.
    *   **_B_** is named as the continue-target for an OpLoopMerge instruction in a reachable block **_H_**.  Additionally there are no branches to **_B_**, and **_B_** contains only an OpLabel and an OpBranch instruction to **_H._**


#### Scope

**Scope** when used for memory must be one of:



*   **Workgroup**
*   **Subgroup**
*   **QueueFamilyKHR**
*   TBD **Invocation? Device?**

**Scope** when used for execution must be one of:



*   **Workgroup**
*   **Subgroup**


#### Memory Semantics

Among mask bits up to and including 0x10 (SequentiallyConsistent), only the following may be set:



*   **Acquire**
*   **Release**
*   **AcquireRelease**  [how does this interact with the memory model?]

The following mask bits may be used in any combination:



*   **UniformMemory**
*   **WorkgroupMemory**
*   **ImageMemory**
*   **OutputMemoryKHR**
*   **MakeAvailableKHR**
*   **MakeVisibleKHR**


#### Memory Access

TBD:  To simplify the programming model: could require if an instruction takes a memory access operand and operates on a storage class other than Function or Private, then it must use NonPrivateKHR.  This could give up some performance.

TBD: Consider a similar simplification for image accesses.


#### Initializers

Variable declarations that include initializers must be for one of the following storage classes:



*   **Output**
*   **Private**
*   **Function**

TBD: Require variables in Output, Private, and Function storage classes to have initializers (or clearly be unconditionally set?).


#### Precision and Accuracy



*   INF, NaN, etc. …. TBD
*   Exceptions are never raised, but signaling NaNs can be generated.
*   Denorms… TBD
*   Rounding… TBD
*   Single precision…. [See tables in Vulkan spec]
*   Double precision is at least the precision of single precision.
*   Error bounds (ULP)
*   … TBD

TBD: Fusing and reassociation of floating point operations is allowed when those instructions are not decorated with **NoContraction**.


### Instructions



*   OpUndef is not allowed.
*   OpVectorShuffle may not have a component literal with value 0xFFFFFFFF.


## Data Types and Layouts


### Images and Samplers

[List all rules about what formats, combinations, types, etc. are used. Note this is mostly SPIR-V independent.]



*   TBD
*   ...


### Interface

Interfaces discussed here are those that are:



*   connecting one shader stage to the subsequent shader stage; from the former's **Output** storage class to the latter's **Input** storage class
*   between the application and the **Input** storage class to the first shader stage in the pipeline
*   between the **Output** storage class of the last tages and the application
*   between the application and the shader stages through buffers: storage buffer blocks and uniform buffer blocks
*   Push constants?

Each such interface must match, via the following matching rules:



*   **Location/Component** … see section 14.1.4 "Location Assignment" in the Vulkan spec.
*   TBD: What does the buffer interface look like?  **Block** decoration and **StorageBuffer**, etc.?
*   TBD: Built-in variables placed in a block go only in one block, with nothing but built-in variables.
*   …
*   TBD, see section 14.1.3 "Interface Matching" from the Vulkan spec.
*   See 14.2. "Vertex Input Interface" in the Vulkan spec.
*   See 14.3. "Fragment Output Interface" in the Vulkan spec.
*   Attachment interface?  14.4
*   Push Constant? 14.5.1
*   See 14.5.2. "Descriptor Set Interface" and 14.5.3. "DescriptorSet and Binding Assignment"


### Layout

All SPIR-V aggregates and matrices appearing in the **Uniform** (... TBD) storage classes must be explicitly and fully laid out through the **Offset**, **ArrayStride**, and **MatrixStride** decorations. These must satisfy the following rules:

[ insert the same rules as given in section 14.5.4. "Offset and Stride Assignment" of the Vulkan spec.] TBD

