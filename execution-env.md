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

### Out-of-bounds Access Behavior

Typically a WebGPU implementation can make sure that the _App_ can only access resources provided in the ExecutionContext by validating the code in the shader module.
However this only guarantees that the correct resources are accessed, not that they are accessed in bounds.

A WebGPU implementation must ensure out of bounds accesses don't cause accesses outside of the ExecutionContext.
This will typically be done by instrumenting the generated code. Because this code is highly performance sensitive, the WebGPU implementation is granted multiple options for enforcing security.

The following rules are based on those defined by the
*robustBufferAccess* feature in the [Vulkan 1.0
specification](https://www.khronos.org/registry/vulkan/specs/1.0/html/vkspec.html#features-features).

Out-of-bounds buffer loads _may_ result in any of the following behaviors:

* Return values from anywhere within the memory range(s) bound to the
  buffer (possibly including bytes of memory past the end of the
  buffer, up to the end of the bound range).

* Return zero values, or _(0,0,0,x)_ vectors for vector reads,
  where x is a valid value represented in the type of the vector
  components and _may_ be any of:

  * 0, 1, or the maximum representable positive integer value, for
    signed or unsigned integer components.

  * 0.0 or 1.0, for floating-point components.

* Cause a trap: that is, cause the shader invocation to write zero values to all shader stage outputs, and then terminate without other side effects.

Out-of-bounds buffer writes _may_ result in any of the following behaviors:

* Be discarded.
* Modify values within the memory range(s) bound to the buffer, but
  _must_ not modify any other memory.
* Cause a trap.

Out-of-bounds atomics _may_ result in any of the following behaviors:

* Be discarded.
* Modify values within the memory range(s) bound to the buffer, but
  _must_ not modify any other memory.
* Return an undefined value.
* Cause a trap.

Additional functionality exists to create pointers to values inside resources.
Creating a pointer for a resource that points outside the resource _may_ result in any of the following behaviors:

* Defer the out of bounds behavior to when the pointer is dereferenced.
* Create a pointer to any location within the resource. (this is equivalent to deferring and "access any location" for resource accesses).
* Cause a trap.

(TODO are there pointers to unsized arrays? Could talk about the "sized part of the type if that's the case" and use the "any location" rule)

## WebGPU Execution Environment Specification for SPIR-V

### Introduction

WebGPU is a client API for SPIR-V. Valid s<span style="color:#222222;">haders for WebGPU are defined by this document in addition to the [Khronos SPIR-V Specification, version 1.3, revision 5](https://www.khronos.org/registry/spir-v/specs/unified1/SPIRV.html)<span style="color:#222222;">.<span style="color:#222222;">  [TBD: update to latest available specification] See the "Version" subsection below for allowed version numbers.</span></span></span>

The following sections visit the parts of a SPIR-V module, in order, covering each section, and state additional requirements for using shaders in WebGPU. These are in addition to all requirements already present in the SPIR-V specification itself.

### Undefined Behaviour

The SPIR-V specification states that certain actions by a SPIR-V shader result in _undefined_ _behaviour_.  A WebGPU implementation is required to be secure.  Therefore in the context of WebGPU, a shader which triggers SPIR-V's undefined behaviour may continue to execute in an arbitrary manner except that it may not sense or mutate any state outside its ExecutionContext, and it may not cause or sense I/O outside its ExecutionContext.

NOTE: One of the possible behaviours of an ill-behaved application is non-termination.  The WebGPU specification should say what happens to a runaway application.

### Opcodes Potentially Resulting in Out-Of-Bounds Accesses

Certain operations in SPIR-V which are valid in the WebGPU environment could result in out-of-bounds accesses. Their behavior is governed by the [out-of-bounds access behavior](#out-of-bounds-access-behavior) section above.

* `OpLoad`, `OpRead`, `OpMemoryCopy` when the pointer is out of bounds.
* `OpAccessChain` when any of the indices for array accesses are larger (or equal) to the array's size or when the pointer is out of bounds.
   Note that the size of arrays are known at runtime for `OpTypeRuntimeArray`.
   Sizes for `OpTypeArray` are always known at pipeline compilation time when all specialization constants have been set.
* `OpInBoundsAccessChain` under the same conditions as `OpAccessChain`, effectively making it an exact equivalent of `OpAccessChain`.
* All operations starting with `OpAtomic` when the pointer is out of bounds.
* `OpImageFetch`, `OpImageRead`, `OpImageWrite` when coordinates aren't in bounds of the resource.
* `OpImageTexelPointer` when the coordinates aren't in bounds of the resource.

Other operations can result in out-of-bounds accesses even if they aren't on resources. The same out of bounds rules apply to them as well:

* `OpVectorIndexDynamic` and `OpVectorInsertDynamic` when the index is larger than, or equal to, the vector size.
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


[//]: # (No **SampledBuffer** or **ImageBuffer**. See https://github.com/gpuweb/gpuweb/issues/162)

*   **Matrix**
*   **Shader**
*   **Sampled1D**
*   **Image1D**
*   **DerivativeControl**
*   **ImageQuery**
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
*   The _Name_ of an **OpEntryPoint**
    * cannot be used by any other **OpEntryPoint** in the same module.
    * has its length bounded due to SPIR-V instruction length limits.
    * may be limited by the WebGPU specification. [WHLSL Issue 292](https://github.com/gpuweb/WHLSL/issues/292)

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

[//]: # (No subgroups support. Useful, but not widely available?)


### Debug Instructions

All debug instructions must be stripped.


### Annotations

OpDecorationGroup, OpGroupDecorate, OpGroupMemberDecorate are not allowed.

Only the following decorations are valid:



[//]: # (No **RelaxedPrecision**: WHLSL doesn't have min16float min10float)
[//]: # (No **Sample**: WebGPU does not support sample rate shading)
[//]: # (No **Invariant**: Assume WebGPU MVP does not support this for complexity reasons)
[//]: # (No **Coherent**: Assume Vulkan memory model)
[//]: # (No **NoSignedWrap**, **NoUnsignedWrap**: Although useful, they introduce undefined behaviour cases)

*   **SpecId**
*   **Block**
*   **RowMajor**
*   **ColMajor**
*   **ArrayStride** (also see "Layouts" below)
*   **MatrixStride** (also see "Layouts" below)
*   **Builtin** (also see "Built-In Variables" below)
*   **NoPerspective**, only in the **Input** storage class and the **Fragment** execution model
*   **Flat**, only in the **Input** storage class and the **Fragment** execution model
*   **Centroid**
*   **Restrict**
*   **Aliased**
*   **NonWritable**
*   **NonReadable**
*   **Uniform**
*   **Location**, required for all **Input** and **Output**, see "Interface" section below
*   **Component**
*   **Index**
*   **Binding**
*   **DescriptorSet**
*   **Offset** (also see "Layouts" below)
*   **NoContraction**


#### Built-In Variables

When decorating with the **Builtin** decoration, only the following are valid, and these are further restricted as indicated by the table.


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
</table>


Full semantic descriptions of what these mean, how the behave, when they have what values, etc. is part of the WebGPU specification proper, not this specification.


### Types

Scalar floating-point types (**OpTypeFloat**) must have one of the following widths:



*   32 bits

Scalar integer types (**OpTypeInteger**) must have one of the following widths:



*   32 bits

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
*   **PushConstant**, if WebGPU supports them: [WebGPU Issue 75](https://github.com/gpuweb/gpuweb/issues/75)
*   **Private**
*   **Function**

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


[//]: # (No **Device** since that's optional for Vulkan memory model.  In single-device configurations QueueFamilyKHR is the same as Device)

*   **Workgroup**
*   **Subgroup**
*   **QueueFamilyKHR**

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

If a variable declaration has an initializer, then the variable must be in one of the following storage classes:

*   **Output**
*   **Private**
*   **Function**

All variables in the following storage classes must have an initializer:

*   **Output**
*   **Private**
*   **Function**


#### Precision and Accuracy



*   INF, NaN, etc. …. TBD
*   Exceptions are never raised, but signaling NaNs can be generated.
*   Denorms… TBD
*   Rounding… TBD
*   Single precision…. [See tables in Vulkan spec]
*   Double precision is at least the precision of single precision.
*   Error bounds (ULP)
*   … TBD

Fusing and reassociation of floating point operations is allowed when those instructions are not decorated with **NoContraction**.


### Instructions



*   OpUndef is not allowed.
*   OpVectorShuffle may not have a component literal with value 0xFFFFFFFF.
*   Supported and unsupported OpCodes are iterated in [Appendix A](#a-supported-opcodes) and [Appendix B](#b-unsupported-opcodes).


## Data Types and Layouts


### Images and Samplers

TBD: List rules for supported formats, combinations, types, etc.  This is mostly independent of SPIR-V.

See [WebGPU Issue 163](https://github.com/gpuweb/gpuweb/issues/163)


### Interface

Interfaces discussed here are those that are:



*   connecting one shader stage to the subsequent shader stage; from the former's **Output** storage class to the latter's **Input** storage class
*   between the application and the **Input** storage class to the first shader stage in the pipeline
*   between the **Output** storage class of the last tages and the application
*   between the application and the shader stages through buffers: storage buffer blocks and uniform buffer blocks
*   Push constants, if WebGPU supports them: [WebGPU Issue 75](https://github.com/gpuweb/gpuweb/issues/75)

Each such interface must match, via the following matching rules:



*   **Location/Component** … see section 14.1.4 "Location Assignment" in the Vulkan spec.
*   TBD: What does the buffer interface look like?  **Block** decoration and **StorageBuffer**, etc.?
*   TBD: Built-in variables placed in a block go only in one block, with nothing but built-in variables.
*   …
*   TBD, see section 14.1.3 "Interface Matching" from the Vulkan spec.
*   See 14.2. "Vertex Input Interface" in the Vulkan spec.
*   See 14.3. "Fragment Output Interface" in the Vulkan spec.
*   Attachment interface?  14.4
*   Push Constant 14.5.1, if WebGPU supports them: [WebGPU Issue 75](https://github.com/gpuweb/gpuweb/issues/75)
*   See 14.5.2. "Descriptor Set Interface" and 14.5.3. "DescriptorSet and Binding Assignment"


### Layout

_Text in this section is adapted from [14.5.4 Offset and Stride Assignment](https://www.khronos.org/registry/vulkan/specs/1.1/html/vkspec.html#interfaces-resources-layout) section in
the [Vulkan Specification version 1.1][Vulkan 1.1]_

All SPIR-V aggregates and matrices appearing in the **Uniform** and **StorageBuffer** storage classes must be explicitly and fully laid out through the **Offset**, **ArrayStride**, **MatrixStride**, **RowMajor**, and **ColMajor** decorations. These must satisfy the following rules described in this section.

Note: If WebGPU supports push constants, then aggregates and matrices in the **PushConstant** storage class must also be explicitly laid out. See [WebGPU Issue 75](https://github.com/gpuweb/gpuweb/issues/75).


Two different layouts requirements are described below: standard uniform buffer layout,
and standard storage buffer layout.  The rules to apply for a specific resource
depends on the type of the buffer resource.

#### General considerations

Offsets and strides are expressed in units of 8-bit bytes.

A scalar numeric value of 32 bits occupies 4 bytes in memory, i.e. its size is 4.
In general a scalar value of _N_ bits occupies _floor((N + 7) / 8)_ bytes.

Layout decorations on SPIR-V objects may not conflict or be duplicated:

* A SPIR-V object or structure member must have at most one decoration each of kind:
   * **Offset**
   * **ArrayStride**
   * **MatrixStride**
   * **RowMajor**
   * **ColMajor**
* A SPIR-V structure member must not be decorated with both **RowMajor** and **ColMajor**.

The numeric order of **Offset** decorations does not need to follow structure member
declaration order. That is, offsets of structure members are not necessarily monotonic.

Section [2.16.2 Validation Rules for Shader Capabilities](https://www.khronos.org/registry/spir-v/specs/unified1/SPIRV.html#_a_id_shadervalidation_a_validation_rules_for_shader_a_href_capability_capabilities_a) of the SPIR-V specification has additional constraints not copied here.
For example, in effect:

* Structure members may not overlap.
* Each matrix in **Uniform** or **StorageBuffer** storage must have
its row- or column-majorness expressed as a **RowMajor** or **ColMajor**
decoration on the nearest enclosing structure member.

#### Standard Uniform Buffer Layout

This section specifies rules for _Standard Uniform Buffer Layout_.

The _base alignment_ of the type of an **OpTypeStruct** member is
defined recursively as follows:

* A scalar of size _N_ bytes has a base alignment of _N_.
* A two-component vector, with components of size _N_ bytes, has a base
  alignment of _2 N_.
* A three- or four-component vector, with components of size _N_ bytes, has
  a base alignment of _4 N_.
* An array has a base alignment equal to the base alignment of its element
  type, rounded up to a multiple of _16_.
* A structure has a base alignment equal to the largest base alignment of
  any of its members, rounded up to a multiple of _16_.
* A row-major matrix of _C_ columns has a base alignment equal to the
  base alignment of a vector of _C_ matrix components.
* A column-major matrix has a base alignment equal to the base alignment
  of the matrix column type.

A member is defined to _improperly straddle_ if either of the following are
true:

* It is a vector with total size less than or equal to 16 bytes, and has
  **Offset** decorations placing its first byte at _F_ and its last
  byte at _L_, where _floor(F / 16) != floor(L / 16)_.
* It is a vector with total size greater than 16 bytes and has its
  **Offset** decorations placing its first byte at a non-integer multiple
  of 16.

Every member of an **OpTypeStruct** with storage class of **Uniform** and
a decoration of **Block** (uniform buffers) must be laid out according to
the following rules:

* The **Offset** decoration of a scalar, an array, a structure, or a
  matrix must be a multiple of its base alignment.
* The **Offset** decoration of a vector must be an integer multiple of
  the base alignment of its scalar component type, and must not
  improperly straddle, as defined above.
* Any **ArrayStride** or **MatrixStride** decoration must be an integer
  multiple of the base alignment of the array or matrix from above.
* The **Offset** decoration of a member must not place it between the
  end of a structure or an array and the next multiple of the base
  alignment of that structure or array.

Note: The **std140** layout in GLSL satisfies these rules.

Note: The above assumes WebGPU can use what Vulkan calls _relaxed block layout_,
which is optional in Vulkan 1.0 but a required feature of Vulkan 1.1.

#### Standard Storage Buffer Layout

This section specifies rules for _Standard Storage Buffer Layout_.

Member variables of an **OpTypeStruct** with a storage class of
**StorageBuffer** with a decoration of **Block**
must be laid out as for _Standard Uniform Buffer Layout_,  except
for array and structure base alignment which do not need to be rounded up to
a multiple of _16_.

Note: The **std430** in GLSL satisfies these rules.

[//]: # (Assume no scalar block aligment)

# References
<a name="vulkan1.1"
href="https://www.khronos.org/registry/vulkan/specs/1.1/html/vkspec.html">Vulkan Specification, version 1.1</a>, The Khronos Group Inc.  Portions copied and modified under Creative Commons Attribution 4.0 International License.  See terms at [Vulkan-Docs/COPYING.md](https://github.com/KhronosGroup/Vulkan-Docs/blob/master/COPYING.md)

[Vulkan 1.1]: #vulkan1.1

# Appendices

## A: Supported OpCodes
<table>
  <tr>
   <td><strong>OpCode</strong>
   </td>
   <td><strong>Name</strong>
   </td>
  </tr>
  <tr>
   <td>0
   </td>
   <td>OpNop
   </td>
  </tr>
  <tr>
   <td>10
   </td>
   <td>OpExtension
   </td>
  </tr>
  <tr>
   <td>11
   </td>
   <td>OpExtInstImport
   </td>
  </tr>
  <tr>
   <td>12
   </td>
   <td>OpExtInst
   </td>
  </tr>
  <tr>
   <td>14
   </td>
   <td>OpMemoryModel
   </td>
  </tr>
  <tr>
   <td>15
   </td>
   <td>OpEntryPoint
   </td>
  </tr>
  <tr>
   <td>16
   </td>
   <td>OpExecutionMode
   </td>
  </tr>
  <tr>
   <td>17
   </td>
   <td>OpCapability
   </td>
  </tr>
  <tr>
   <td>19
   </td>
   <td>OpTypeVoid
   </td>
  </tr>
  <tr>
   <td>20
   </td>
   <td>OpTypeBool
   </td>
  </tr>
  <tr>
   <td>21
   </td>
   <td>OpTypeInt
   </td>
  </tr>
  <tr>
   <td>22
   </td>
   <td>OpTypeFloat
   </td>
  </tr>
  <tr>
   <td>23
   </td>
   <td>OpTypeVector
   </td>
  </tr>
  <tr>
   <td>24
   </td>
   <td>OpTypeMatrix
   </td>
  </tr>
  <tr>
   <td>25
   </td>
   <td>OpTypeImage
   </td>
  </tr>
  <tr>
   <td>26
   </td>
   <td>OpTypeSampler
   </td>
  </tr>
  <tr>
   <td>27
   </td>
   <td>OpTypeSampledImage
   </td>
  </tr>
  <tr>
   <td>28
   </td>
   <td>OpTypeArray
   </td>
  </tr>
  <tr>
   <td>29
   </td>
   <td>OpTypeRuntimeArray
   </td>
  </tr>
  <tr>
   <td>30
   </td>
   <td>OpTypeStruct
   </td>
  </tr>
  <tr>
   <td>32
   </td>
   <td>OpTypePointer
   </td>
  </tr>
  <tr>
   <td>33
   </td>
   <td>OpTypeFunction
   </td>
  </tr>
  <tr>
   <td>41
   </td>
   <td>OpConstantTrue
   </td>
  </tr>
  <tr>
   <td>42
   </td>
   <td>OpConstantFalse
   </td>
  </tr>
  <tr>
   <td>43
   </td>
   <td>OpConstant
   </td>
  </tr>
  <tr>
   <td>44
   </td>
   <td>OpConstantComposite
   </td>
  </tr>
  <tr>
   <td>46
   </td>
   <td>OpConstantNull
   </td>
  </tr>
  <tr>
   <td>48
   </td>
   <td>OpSpecConstantTrue
   </td>
  </tr>
  <tr>
   <td>49
   </td>
   <td>OpSpecConstantFalse
   </td>
  </tr>
  <tr>
   <td>50
   </td>
   <td>OpSpecConstant
   </td>
  </tr>
  <tr>
   <td>51
   </td>
   <td>OpSpecConstantComposite
   </td>
  </tr>
  <tr>
   <td>52
   </td>
   <td>OpSpecConstantOp
   </td>
  </tr>
  <tr>
   <td>54
   </td>
   <td>OpFunction
   </td>
  </tr>
  <tr>
   <td>55
   </td>
   <td>OpFunctionParameter
   </td>
  </tr>
  <tr>
   <td>56
   </td>
   <td>OpFunctionEnd
   </td>
  </tr>
  <tr>
   <td>57
   </td>
   <td>OpFunctionCall
   </td>
  </tr>
  <tr>
   <td>59
   </td>
   <td>OpVariable
   </td>
  </tr>
  <tr>
   <td>60
   </td>
   <td>OpImageTexelPointer
   </td>
  </tr>
  <tr>
   <td>61
   </td>
   <td>OpLoad
   </td>
  </tr>
  <tr>
   <td>62
   </td>
   <td>OpStore
   </td>
  </tr>
  <tr>
   <td>63
   </td>
   <td>OpCopyMemory
   </td>
  </tr>
  <tr>
   <td>65
   </td>
   <td>OpAccessChain
   </td>
  </tr>
  <tr>
   <td>66
   </td>
   <td>OpInBoundsAccessChain
   </td>
  </tr>
  <tr>
   <td>68
   </td>
   <td>OpArrayLength
   </td>
  </tr>
  <tr>
   <td>71
   </td>
   <td>OpDecorate
   </td>
  </tr>
  <tr>
   <td>72
   </td>
   <td>OpMemberDecorate
   </td>
  </tr>
  <tr>
   <td>77
   </td>
   <td>OpVectorExtractDynamic
   </td>
  </tr>
  <tr>
   <td>78
   </td>
   <td>OpVectorInsertDynamic
   </td>
  </tr>
  <tr>
   <td>79
   </td>
   <td>OpVectorShuffle
   </td>
  </tr>
  <tr>
   <td>80
   </td>
   <td>OpCompositeConstruct
   </td>
  </tr>
  <tr>
   <td>81
   </td>
   <td>OpCompositeExtract
   </td>
  </tr>
  <tr>
   <td>82
   </td>
   <td>OpCompositeInsert
   </td>
  </tr>
  <tr>
   <td>83
   </td>
   <td>OpCopyObject
   </td>
  </tr>
  <tr>
   <td>84
   </td>
   <td>OpTranspose
   </td>
  </tr>
  <tr>
   <td>86
   </td>
   <td>OpSampledImage
   </td>
  </tr>
  <tr>
   <td>87
   </td>
   <td>OpImageSampleImplicitLod
   </td>
  </tr>
  <tr>
   <td>88
   </td>
   <td>OpImageSampleExplicitLod
   </td>
  </tr>
  <tr>
   <td>89
   </td>
   <td>OpImageSampleDrefImplicitLod
   </td>
  </tr>
  <tr>
   <td>90
   </td>
   <td>OpImageSampleDrefExplicitLod
   </td>
  </tr>
  <tr>
   <td>91
   </td>
   <td>OpImageSampleProjImplicitLod
   </td>
  </tr>
  <tr>
   <td>92
   </td>
   <td>OpImageSampleProjExplicitLod
   </td>
  </tr>
  <tr>
   <td>93
   </td>
   <td>OpImageSampleProjDrefImplicitLod
   </td>
  </tr>
  <tr>
   <td>94
   </td>
   <td>OpImageSampleProjDrefExplicitLod
   </td>
  </tr>
  <tr>
   <td>95
   </td>
   <td>OpImageFetch
   </td>
  </tr>
  <tr>
   <td>96
   </td>
   <td>OpImageGather
   </td>
  </tr>
  <tr>
   <td>97
   </td>
   <td>OpImageDrefGather
   </td>
  </tr>
  <tr>
   <td>98
   </td>
   <td>OpImageRead
   </td>
  </tr>
  <tr>
   <td>99
   </td>
   <td>OpImageWrite
   </td>
  </tr>
  <tr>
   <td>100
   </td>
   <td>OpImage
   </td>
  </tr>
  <tr>
   <td>105
   </td>
   <td>OpImageQueryLod
   </td>
  </tr>
  <tr>
   <td>109
   </td>
   <td>OpConvertFToU
   </td>
  </tr>
  <tr>
   <td>110
   </td>
   <td>OpConvertFToS
   </td>
  </tr>
  <tr>
   <td>111
   </td>
   <td>OpConvertSToF
   </td>
  </tr>
  <tr>
   <td>112
   </td>
   <td>OpConvertUToF
   </td>
  </tr>
  <tr>
   <td>113
   </td>
   <td>OpUConvert
   </td>
  </tr>
  <tr>
   <td>114
   </td>
   <td>OpSConvert
   </td>
  </tr>
  <tr>
   <td>115
   </td>
   <td>OpFConvert
   </td>
  </tr>
  <tr>
   <td>116
   </td>
   <td>OpQuantizeToF16
   </td>
  </tr>
  <tr>
   <td>124
   </td>
   <td>OpBitcast
   </td>
  </tr>
  <tr>
   <td>126
   </td>
   <td>OpSNegate
   </td>
  </tr>
  <tr>
   <td>127
   </td>
   <td>OpFNegate
   </td>
  </tr>
  <tr>
   <td>128
   </td>
   <td>OpIAdd
   </td>
  </tr>
  <tr>
   <td>129
   </td>
   <td>OpFAdd
   </td>
  </tr>
  <tr>
   <td>130
   </td>
   <td>OpISub
   </td>
  </tr>
  <tr>
   <td>131
   </td>
   <td>OpFSub
   </td>
  </tr>
  <tr>
   <td>132
   </td>
   <td>OpIMul
   </td>
  </tr>
  <tr>
   <td>133
   </td>
   <td>OpFMul
   </td>
  </tr>
  <tr>
   <td>134
   </td>
   <td>OpUDiv
   </td>
  </tr>
  <tr>
   <td>135
   </td>
   <td>OpSDiv
   </td>
  </tr>
  <tr>
   <td>136
   </td>
   <td>OpFDiv
   </td>
  </tr>
  <tr>
   <td>137
   </td>
   <td>OpUMod
   </td>
  </tr>
  <tr>
   <td>138
   </td>
   <td>OpSRem
   </td>
  </tr>
  <tr>
   <td>139
   </td>
   <td>OpSMod
   </td>
  </tr>
  <tr>
   <td>140
   </td>
   <td>OpFRem
   </td>
  </tr>
  <tr>
   <td>141
   </td>
   <td>OpFMod
   </td>
  </tr>
  <tr>
   <td>142
   </td>
   <td>OpVectorTimesScalar
   </td>
  </tr>
  <tr>
   <td>143
   </td>
   <td>OpMatrixTimesScalar
   </td>
  </tr>
  <tr>
   <td>144
   </td>
   <td>OpVectorTimesMatrix
   </td>
  </tr>
  <tr>
   <td>145
   </td>
   <td>OpMatrixTimesVector
   </td>
  </tr>
  <tr>
   <td>146
   </td>
   <td>OpMatrixTimesMatrix
   </td>
  </tr>
  <tr>
   <td>147
   </td>
   <td>OpOuterProduct
   </td>
  </tr>
  <tr>
   <td>148
   </td>
   <td>OpDot
   </td>
  </tr>
  <tr>
   <td>149
   </td>
   <td>OpIAddCarry
   </td>
  </tr>
  <tr>
   <td>150
   </td>
   <td>OpISubBorrow
   </td>
  </tr>
  <tr>
   <td>151
   </td>
   <td>OpUMulExtended
   </td>
  </tr>
  <tr>
   <td>152
   </td>
   <td>OpSMulExtended
   </td>
  </tr>
  <tr>
   <td>154
   </td>
   <td>OpAny
   </td>
  </tr>
  <tr>
   <td>155
   </td>
   <td>OpAll
   </td>
  </tr>
  <tr>
   <td>156
   </td>
   <td>OpIsNan
   </td>
  </tr>
  <tr>
   <td>157
   </td>
   <td>OpIsInf
   </td>
  </tr>
  <tr>
   <td>164
   </td>
   <td>OpLogicalEqual
   </td>
  </tr>
  <tr>
   <td>165
   </td>
   <td>OpLogicalNotEqual
   </td>
  </tr>
  <tr>
   <td>166
   </td>
   <td>OpLogicalOr
   </td>
  </tr>
  <tr>
   <td>167
   </td>
   <td>OpLogicalAnd
   </td>
  </tr>
  <tr>
   <td>168
   </td>
   <td>OpLogicalNot
   </td>
  </tr>
  <tr>
   <td>169
   </td>
   <td>OpSelect
   </td>
  </tr>
  <tr>
   <td>170
   </td>
   <td>OpIEqual
   </td>
  </tr>
  <tr>
   <td>171
   </td>
   <td>OpINotEqual
   </td>
  </tr>
  <tr>
   <td>172
   </td>
   <td>OpUGreaterThan
   </td>
  </tr>
  <tr>
   <td>173
   </td>
   <td>OpSGreaterThan
   </td>
  </tr>
  <tr>
   <td>174
   </td>
   <td>OpUGreaterThanEqual
   </td>
  </tr>
  <tr>
   <td>175
   </td>
   <td>OpSGreaterThanEqual
   </td>
  </tr>
  <tr>
   <td>176
   </td>
   <td>OpULessThan
   </td>
  </tr>
  <tr>
   <td>177
   </td>
   <td>OpSLessThan
   </td>
  </tr>
  <tr>
   <td>178
   </td>
   <td>OpULessThanEqual
   </td>
  </tr>
  <tr>
   <td>179
   </td>
   <td>OpSLessThanEqual
   </td>
  </tr>
  <tr>
   <td>180
   </td>
   <td>OpFOrdEqual
   </td>
  </tr>
  <tr>
   <td>181
   </td>
   <td>OpFUnordEqual
   </td>
  </tr>
  <tr>
   <td>182
   </td>
   <td>OpFOrdNotEqual
   </td>
  </tr>
  <tr>
   <td>183
   </td>
   <td>OpFUnordNotEqual
   </td>
  </tr>
  <tr>
   <td>184
   </td>
   <td>OpFOrdLessThan
   </td>
  </tr>
  <tr>
   <td>185
   </td>
   <td>OpFUnordLessThan
   </td>
  </tr>
  <tr>
   <td>186
   </td>
   <td>OpFOrdGreaterThan
   </td>
  </tr>
  <tr>
   <td>187
   </td>
   <td>OpFUnordGreaterThan
   </td>
  </tr>
  <tr>
   <td>188
   </td>
   <td>OpFOrdLessThanEqual
   </td>
  </tr>
  <tr>
   <td>189
   </td>
   <td>OpFUnordLessThanEqual
   </td>
  </tr>
  <tr>
   <td>190
   </td>
   <td>OpFOrdGreaterThanEqual
   </td>
  </tr>
  <tr>
   <td>191
   </td>
   <td>OpFUnordGreaterThanEqual
   </td>
  </tr>
  <tr>
   <td>194
   </td>
   <td>OpShiftRightLogical
   </td>
  </tr>
  <tr>
   <td>195
   </td>
   <td>OpShiftRightArithmetic
   </td>
  </tr>
  <tr>
   <td>196
   </td>
   <td>OpShiftLeftLogical
   </td>
  </tr>
  <tr>
   <td>197
   </td>
   <td>OpBitwiseOr
   </td>
  </tr>
  <tr>
   <td>198
   </td>
   <td>OpBitwiseXor
   </td>
  </tr>
  <tr>
   <td>199
   </td>
   <td>OpBitwiseAnd
   </td>
  </tr>
  <tr>
   <td>200
   </td>
   <td>OpNot
   </td>
  </tr>
  <tr>
   <td>201
   </td>
   <td>OpBitFieldInsert
   </td>
  </tr>
  <tr>
   <td>202
   </td>
   <td>OpBitFieldSExtract
   </td>
  </tr>
  <tr>
   <td>203
   </td>
   <td>OpBitFieldUExtract
   </td>
  </tr>
  <tr>
   <td>204
   </td>
   <td>OpBitReverse
   </td>
  </tr>
  <tr>
   <td>205
   </td>
   <td>OpBitCount
   </td>
  </tr>
  <tr>
   <td>207
   </td>
   <td>OpDPdx
   </td>
  </tr>
  <tr>
   <td>208
   </td>
   <td>OpDPdy
   </td>
  </tr>
  <tr>
   <td>209
   </td>
   <td>OpFwidth
   </td>
  </tr>
  <tr>
   <td>210
   </td>
   <td>OpDPdxFine
   </td>
  </tr>
  <tr>
   <td>211
   </td>
   <td>OpDPdyFine
   </td>
  </tr>
  <tr>
   <td>212
   </td>
   <td>OpFwidthFine
   </td>
  </tr>
  <tr>
   <td>213
   </td>
   <td>OpDPdxCoarse
   </td>
  </tr>
  <tr>
   <td>214
   </td>
   <td>OpDPdyCoarse
   </td>
  </tr>
  <tr>
   <td>215
   </td>
   <td>OpFwidthCoarse
   </td>
  </tr>
  <tr>
   <td>224
   </td>
   <td>OpControlBarrier
   </td>
  </tr>
  <tr>
   <td>225
   </td>
   <td>OpMemoryBarrier
   </td>
  </tr>
  <tr>
   <td>227
   </td>
   <td>OpAtomicLoad
   </td>
  </tr>
  <tr>
   <td>228
   </td>
   <td>OpAtomicStore
   </td>
  </tr>
  <tr>
   <td>229
   </td>
   <td>OpAtomicExchange
   </td>
  </tr>
  <tr>
   <td>230
   </td>
   <td>OpAtomicCompareExchange
   </td>
  </tr>
  <tr>
   <td>232
   </td>
   <td>OpAtomicIIncrement
   </td>
  </tr>
  <tr>
   <td>233
   </td>
   <td>OpAtomicIDecrement
   </td>
  </tr>
  <tr>
   <td>234
   </td>
   <td>OpAtomicIAdd
   </td>
  </tr>
  <tr>
   <td>235
   </td>
   <td>OpAtomicISub
   </td>
  </tr>
  <tr>
   <td>236
   </td>
   <td>OpAtomicSMin
   </td>
  </tr>
  <tr>
   <td>237
   </td>
   <td>OpAtomicUMin
   </td>
  </tr>
  <tr>
   <td>238
   </td>
   <td>OpAtomicSMax
   </td>
  </tr>
  <tr>
   <td>239
   </td>
   <td>OpAtomicUMax
   </td>
  </tr>
  <tr>
   <td>240
   </td>
   <td>OpAtomicAnd
   </td>
  </tr>
  <tr>
   <td>241
   </td>
   <td>OpAtomicOr
   </td>
  </tr>
  <tr>
   <td>242
   </td>
   <td>OpAtomicXor
   </td>
  </tr>
  <tr>
   <td>245
   </td>
   <td>OpPhi
   </td>
  </tr>
  <tr>
   <td>246
   </td>
   <td>OpLoopMerge
   </td>
  </tr>
  <tr>
   <td>247
   </td>
   <td>OpSelectionMerge
   </td>
  </tr>
  <tr>
   <td>248
   </td>
   <td>OpLabel
   </td>
  </tr>
  <tr>
   <td>249
   </td>
   <td>OpBranch
   </td>
  </tr>
  <tr>
   <td>250
   </td>
   <td>OpBranchConditional
   </td>
  </tr>
  <tr>
   <td>251
   </td>
   <td>OpSwitch
   </td>
  </tr>
  <tr>
   <td>252
   </td>
   <td>OpKill
   </td>
  </tr>
  <tr>
   <td>253
   </td>
   <td>OpReturn
   </td>
  </tr>
  <tr>
   <td>254
   </td>
   <td>OpReturnValue
   </td>
  </tr>
  <tr>
   <td>255
   </td>
   <td>OpUnreachable
   </td>
  </tr>
  <tr>
   <td>331
   </td>
   <td>OpExecutionModeId
   </td>
  </tr>
  <tr>
   <td>332
   </td>
   <td>OpDecorateId
   </td>
  </tr>
</table>

## B: Unsupported OpCodes
<table>
  <tr>
   <td><strong>OpCode</strong>
   </td>
   <td><strong>Name</strong>
   </td>
  </tr>
  <tr>
   <td>1
   </td>
   <td>OpUndef
   </td>
  </tr>
  <tr>
   <td>2
   </td>
   <td>OpSourceContinued
   </td>
  </tr>
  <tr>
   <td>3
   </td>
   <td>OpSource
   </td>
  </tr>
  <tr>
   <td>4
   </td>
   <td>OpSourceExtension
   </td>
  </tr>
  <tr>
   <td>5
   </td>
   <td>OpName
   </td>
  </tr>
  <tr>
   <td>6
   </td>
   <td>OpMemberName
   </td>
  </tr>
  <tr>
   <td>7
   </td>
   <td>OpString
   </td>
  </tr>
  <tr>
   <td>8
   </td>
   <td>OpLine
   </td>
  </tr>
  <tr>
   <td>31
   </td>
   <td>OpTypeOpaque
   </td>
  </tr>
  <tr>
   <td>34
   </td>
   <td>OpTypeEvent
   </td>
  </tr>
  <tr>
   <td>35
   </td>
   <td>OpTypeDeviceEvent
   </td>
  </tr>
  <tr>
   <td>36
   </td>
   <td>OpTypeReserveId
   </td>
  </tr>
  <tr>
   <td>37
   </td>
   <td>OpTypeQueue
   </td>
  </tr>
  <tr>
   <td>38
   </td>
   <td>OpTypePipe
   </td>
  </tr>
  <tr>
   <td>39
   </td>
   <td>OpTypeForwardPointer
   </td>
  </tr>
  <tr>
   <td>45
   </td>
   <td>OpConstantSampler
   </td>
  </tr>
  <tr>
   <td>64
   </td>
   <td>OpCopyMemorySized
   </td>
  </tr>
  <tr>
   <td>67
   </td>
   <td>OpPtrAccessChain
   </td>
  </tr>
  <tr>
   <td>69
   </td>
   <td>OpGenericPtrMemSemantics
   </td>
  </tr>
  <tr>
   <td>70
   </td>
   <td>OpInBoundsPtrAccessChain
   </td>
  </tr>
  <tr>
   <td>73
   </td>
   <td>OpDecorationGroup
   </td>
  </tr>
  <tr>
   <td>74
   </td>
   <td>OpGroupDecorate
   </td>
  </tr>
  <tr>
   <td>75
   </td>
   <td>OpGroupMemberDecorate
   </td>
  </tr>
  <tr>
   <td>101
   </td>
   <td>OpImageQueryFormat
   </td>
  </tr>
  <tr>
   <td>102
   </td>
   <td>OpImageQueryOrder
   </td>
  </tr>
  <tr>
   <td>103
   </td>
   <td>OpImageQuerySizeLod
   </td>
  </tr>
  <tr>
   <td>104
   </td>
   <td>OpImageQuerySize
   </td>
  </tr>
  <tr>
   <td>106
   </td>
   <td>OpImageQueryLevels
   </td>
  </tr>
  <tr>
   <td>107
   </td>
   <td>OpImageQuerySamples
   </td>
  </tr>
  <tr>
   <td>117
   </td>
   <td>OpConvertPtrToU
   </td>
  </tr>
  <tr>
   <td>118
   </td>
   <td>OpSatConvertSToU
   </td>
  </tr>
  <tr>
   <td>119
   </td>
   <td>OpSatConvertUToS
   </td>
  </tr>
  <tr>
   <td>120
   </td>
   <td>OpConvertUToPtr
   </td>
  </tr>
  <tr>
   <td>121
   </td>
   <td>OpPtrCastToGeneric
   </td>
  </tr>
  <tr>
   <td>122
   </td>
   <td>OpGenericCastToPtr
   </td>
  </tr>
  <tr>
   <td>123
   </td>
   <td>OpGenericCastToPtrExplicit
   </td>
  </tr>
  <tr>
   <td>158
   </td>
   <td>OpIsFinite
   </td>
  </tr>
  <tr>
   <td>159
   </td>
   <td>OpIsNormal
   </td>
  </tr>
  <tr>
   <td>160
   </td>
   <td>OpSignBitSet
   </td>
  </tr>
  <tr>
   <td>161
   </td>
   <td>OpLessOrGreater
   </td>
  </tr>
  <tr>
   <td>162
   </td>
   <td>OpOrdered
   </td>
  </tr>
  <tr>
   <td>163
   </td>
   <td>OpUnordered
   </td>
  </tr>
  <tr>
   <td>218
   </td>
   <td>OpEmitVertex
   </td>
  </tr>
  <tr>
   <td>219
   </td>
   <td>OpEndPrimitive
   </td>
  </tr>
  <tr>
   <td>220
   </td>
   <td>OpEmitStreamVertex
   </td>
  </tr>
  <tr>
   <td>221
   </td>
   <td>OpEndStreamPrimitive
   </td>
  </tr>
  <tr>
   <td>231
   </td>
   <td>OpAtomicCompareExchangeWeak
   </td>
  </tr>
  <tr>
   <td>256
   </td>
   <td>OpLifetimeStart
   </td>
  </tr>
  <tr>
   <td>257
   </td>
   <td>OpLifetimeStop
   </td>
  </tr>
  <tr>
   <td>259
   </td>
   <td>OpGroupAsyncCopy
   </td>
  </tr>
  <tr>
   <td>260
   </td>
   <td>OpGroupWaitEvents
   </td>
  </tr>
  <tr>
   <td>261
   </td>
   <td>OpGroupAll
   </td>
  </tr>
  <tr>
   <td>262
   </td>
   <td>OpGroupAny
   </td>
  </tr>
  <tr>
   <td>263
   </td>
   <td>OpGroupBroadcast
   </td>
  </tr>
  <tr>
   <td>264
   </td>
   <td>OpGroupIAdd
   </td>
  </tr>
  <tr>
   <td>265
   </td>
   <td>OpGroupFAdd
   </td>
  </tr>
  <tr>
   <td>266
   </td>
   <td>OpGroupFMin
   </td>
  </tr>
  <tr>
   <td>267
   </td>
   <td>OpGroupUMin
   </td>
  </tr>
  <tr>
   <td>268
   </td>
   <td>OpGroupSMin
   </td>
  </tr>
  <tr>
   <td>269
   </td>
   <td>OpGroupFMax
   </td>
  </tr>
  <tr>
   <td>270
   </td>
   <td>OpGroupUMax
   </td>
  </tr>
  <tr>
   <td>271
   </td>
   <td>OpGroupSMax
   </td>
  </tr>
  <tr>
   <td>274
   </td>
   <td>OpReadPipe
   </td>
  </tr>
  <tr>
   <td>275
   </td>
   <td>OpWritePipe
   </td>
  </tr>
  <tr>
   <td>276
   </td>
   <td>OpReservedReadPipe
   </td>
  </tr>
  <tr>
   <td>277
   </td>
   <td>OpReservedWritePipe
   </td>
  </tr>
  <tr>
   <td>278
   </td>
   <td>OpReserveReadPipePackets
   </td>
  </tr>
  <tr>
   <td>279
   </td>
   <td>OpReserveWritePipePackets
   </td>
  </tr>
  <tr>
   <td>280
   </td>
   <td>OpCommitReadPipe
   </td>
  </tr>
  <tr>
   <td>281
   </td>
   <td>OpCommitWritePipe
   </td>
  </tr>
  <tr>
   <td>282
   </td>
   <td>OpIsValidReserveId
   </td>
  </tr>
  <tr>
   <td>283
   </td>
   <td>OpGetNumPipePackets
   </td>
  </tr>
  <tr>
   <td>284
   </td>
   <td>OpGetMaxPipePackets
   </td>
  </tr>
  <tr>
   <td>285
   </td>
   <td>OpGroupReserveReadPipePackets
   </td>
  </tr>
  <tr>
   <td>286
   </td>
   <td>OpGroupReserveWritePipePackets
   </td>
  </tr>
  <tr>
   <td>287
   </td>
   <td>OpGroupCommitReadPipe
   </td>
  </tr>
  <tr>
   <td>288
   </td>
   <td>OpGroupCommitWritePipe
   </td>
  </tr>
  <tr>
   <td>291
   </td>
   <td>OpEnqueueMarker
   </td>
  </tr>
  <tr>
   <td>292
   </td>
   <td>OpEnqueueKernel
   </td>
  </tr>
  <tr>
   <td>293
   </td>
   <td>OpGetKernelNDrangeSubGroupCount
   </td>
  </tr>
  <tr>
   <td>294
   </td>
   <td>OpGetKernelNDrangeMaxSubGroupSize
   </td>
  </tr>
  <tr>
   <td>295
   </td>
   <td>OpGetKernelWorkGroupSize
   </td>
  </tr>
  <tr>
   <td>296
   </td>
   <td>OpGetKernelPreferredWorkGroupSizeMultiple
   </td>
  </tr>
  <tr>
   <td>297
   </td>
   <td>OpRetainEvent
   </td>
  </tr>
  <tr>
   <td>298
   </td>
   <td>OpReleaseEvent
   </td>
  </tr>
  <tr>
   <td>299
   </td>
   <td>OpCreateUserEvent
   </td>
  </tr>
  <tr>
   <td>300
   </td>
   <td>OpIsValidEvent
   </td>
  </tr>
  <tr>
   <td>301
   </td>
   <td>OpSetUserEventStatus
   </td>
  </tr>
  <tr>
   <td>302
   </td>
   <td>OpCaptureEventProfilingInfo
   </td>
  </tr>
  <tr>
   <td>303
   </td>
   <td>OpGetDefaultQueue
   </td>
  </tr>
  <tr>
   <td>304
   </td>
   <td>OpBuildNDRange
   </td>
  </tr>
  <tr>
   <td>305
   </td>
   <td>OpImageSparseSampleImplicitLod
   </td>
  </tr>
  <tr>
   <td>306
   </td>
   <td>OpImageSparseSampleExplicitLod
   </td>
  </tr>
  <tr>
   <td>307
   </td>
   <td>OpImageSparseSampleDrefImplicitLod
   </td>
  </tr>
  <tr>
   <td>308
   </td>
   <td>OpImageSparseSampleDrefExplicitLod
   </td>
  </tr>
  <tr>
   <td>309
   </td>
   <td>OpImageSparseSampleProjImplicitLod
   </td>
  </tr>
  <tr>
   <td>310
   </td>
   <td>OpImageSparseSampleProjExplicitLod
   </td>
  </tr>
  <tr>
   <td>311
   </td>
   <td>OpImageSparseSampleProjDrefImplicitLod
   </td>
  </tr>
  <tr>
   <td>312
   </td>
   <td>OpImageSparseSampleProjDrefExplicitLod
   </td>
  </tr>
  <tr>
   <td>313
   </td>
   <td>OpImageSparseFetch
   </td>
  </tr>
  <tr>
   <td>314
   </td>
   <td>OpImageSparseGather
   </td>
  </tr>
  <tr>
   <td>315
   </td>
   <td>OpImageSparseDrefGather
   </td>
  </tr>
  <tr>
   <td>316
   </td>
   <td>OpImageSparseTexelsResident
   </td>
  </tr>
  <tr>
   <td>317
   </td>
   <td>OpNoLine
   </td>
  </tr>
  <tr>
   <td>318
   </td>
   <td>OpAtomicFlagTestAndSet
   </td>
  </tr>
  <tr>
   <td>319
   </td>
   <td>OpAtomicFlagClear
   </td>
  </tr>
  <tr>
   <td>320
   </td>
   <td>OpImageSparseRead
   </td>
  </tr>
  <tr>
   <td>321
   </td>
   <td>OpSizeOf
   </td>
  </tr>
  <tr>
   <td>322
   </td>
   <td>OpTypePipeStorage
   </td>
  </tr>
  <tr>
   <td>323
   </td>
   <td>OpConstantPipeStorage
   </td>
  </tr>
  <tr>
   <td>324
   </td>
   <td>OpCreatePipeFromPipeStorage
   </td>
  </tr>
  <tr>
   <td>325
   </td>
   <td>OpGetKernelLocalSizeForSubgroupCount
   </td>
  </tr>
  <tr>
   <td>326
   </td>
   <td>OpGetKernelMaxNumSubgroups
   </td>
  </tr>
  <tr>
   <td>327
   </td>
   <td>OpTypeNamedBarrier
   </td>
  </tr>
  <tr>
   <td>328
   </td>
   <td>OpNamedBarrierInitialize
   </td>
  </tr>
  <tr>
   <td>329
   </td>
   <td>OpMemoryNamedBarrier
   </td>
  </tr>
  <tr>
   <td>330
   </td>
   <td>OpModuleProcessed
   </td>
  </tr>
  <tr>
   <td>333
   </td>
   <td>OpGroupNonUniformElect
   </td>
  </tr>
  <tr>
   <td>334
   </td>
   <td>OpGroupNonUniformAll
   </td>
  </tr>
  <tr>
   <td>335
   </td>
   <td>OpGroupNonUniformAny
   </td>
  </tr>
  <tr>
   <td>336
   </td>
   <td>OpGroupNonUniformAllEqual
   </td>
  </tr>
  <tr>
   <td>337
   </td>
   <td>OpGroupNonUniformBroadcast
   </td>
  </tr>
  <tr>
   <td>338
   </td>
   <td>OpGroupNonUniformBroadcastFirst
   </td>
  </tr>
  <tr>
   <td>339
   </td>
   <td>OpGroupNonUniformBallot
   </td>
  </tr>
  <tr>
   <td>340
   </td>
   <td>OpGroupNonUniformInverseBallot
   </td>
  </tr>
  <tr>
   <td>341
   </td>
   <td>OpGroupNonUniformBallotBitExtract
   </td>
  </tr>
  <tr>
   <td>342
   </td>
   <td>OpGroupNonUniformBallotBitCount
   </td>
  </tr>
  <tr>
   <td>343
   </td>
   <td>OpGroupNonUniformBallotFindLSB
   </td>
  </tr>
  <tr>
   <td>344
   </td>
   <td>OpGroupNonUniformBallotFindMSB
   </td>
  </tr>
  <tr>
   <td>345
   </td>
   <td>OpGroupNonUniformShuffle
   </td>
  </tr>
  <tr>
   <td>346
   </td>
   <td>OpGroupNonUniformShuffleXor
   </td>
  </tr>
  <tr>
   <td>347
   </td>
   <td>OpGroupNonUniformShuffleUp
   </td>
  </tr>
  <tr>
   <td>348
   </td>
   <td>OpGroupNonUniformShuffleDown
   </td>
  </tr>
  <tr>
   <td>349
   </td>
   <td>OpGroupNonUniformIAdd
   </td>
  </tr>
  <tr>
   <td>350
   </td>
   <td>OpGroupNonUniformFAdd
   </td>
  </tr>
  <tr>
   <td>351
   </td>
   <td>OpGroupNonUniformIMul
   </td>
  </tr>
  <tr>
   <td>352
   </td>
   <td>OpGroupNonUniformFMul
   </td>
  </tr>
  <tr>
   <td>353
   </td>
   <td>OpGroupNonUniformSMin
   </td>
  </tr>
  <tr>
   <td>354
   </td>
   <td>OpGroupNonUniformUMin
   </td>
  </tr>
  <tr>
   <td>355
   </td>
   <td>OpGroupNonUniformFMin
   </td>
  </tr>
  <tr>
   <td>356
   </td>
   <td>OpGroupNonUniformSMax
   </td>
  </tr>
  <tr>
   <td>357
   </td>
   <td>OpGroupNonUniformUMax
   </td>
  </tr>
  <tr>
   <td>358
   </td>
   <td>OpGroupNonUniformFMax
   </td>
  </tr>
  <tr>
   <td>359
   </td>
   <td>OpGroupNonUniformBitwiseAnd
   </td>
  </tr>
  <tr>
   <td>360
   </td>
   <td>OpGroupNonUniformBitwiseOr
   </td>
  </tr>
  <tr>
   <td>361
   </td>
   <td>OpGroupNonUniformBitwiseXor
   </td>
  </tr>
  <tr>
   <td>362
   </td>
   <td>OpGroupNonUniformLogicalAnd
   </td>
  </tr>
  <tr>
   <td>363
   </td>
   <td>OpGroupNonUniformLogicalOr
   </td>
  </tr>
  <tr>
   <td>364
   </td>
   <td>OpGroupNonUniformLogicalXor
   </td>
  </tr>
  <tr>
   <td>365
   </td>
   <td>OpGroupNonUniformQuadBroadcast
   </td>
  </tr>
  <tr>
   <td>366
   </td>
   <td>OpGroupNonUniformQuadSwap
   </td>
  </tr>
  <tr>
   <td>4421
   </td>
   <td>OpSubgroupBallotKHR
   </td>
  </tr>
  <tr>
   <td>4422
   </td>
   <td>OpSubgroupFirstInvocationKHR
   </td>
  </tr>
  <tr>
   <td>4428
   </td>
   <td>OpSubgroupAllKHR
   </td>
  </tr>
  <tr>
   <td>4429
   </td>
   <td>OpSubgroupAnyKHR
   </td>
  </tr>
  <tr>
   <td>4430
   </td>
   <td>OpSubgroupAllEqualKHR
   </td>
  </tr>
  <tr>
   <td>4432
   </td>
   <td>OpSubgroupReadInvocationKHR
   </td>
  </tr>
  <tr>
   <td>5000
   </td>
   <td>OpGroupIAddNonUniformAMD
   </td>
  </tr>
  <tr>
   <td>5001
   </td>
   <td>OpGroupFAddNonUniformAMD
   </td>
  </tr>
  <tr>
   <td>5002
   </td>
   <td>OpGroupFMinNonUniformAMD
   </td>
  </tr>
  <tr>
   <td>5003
   </td>
   <td>OpGroupUMinNonUniformAMD
   </td>
  </tr>
  <tr>
   <td>5004
   </td>
   <td>OpGroupSMinNonUniformAMD
   </td>
  </tr>
  <tr>
   <td>5005
   </td>
   <td>OpGroupFMaxNonUniformAMD
   </td>
  </tr>
  <tr>
   <td>5006
   </td>
   <td>OpGroupUMaxNonUniformAMD
   </td>
  </tr>
  <tr>
   <td>5007
   </td>
   <td>OpGroupSMaxNonUniformAMD
   </td>
  </tr>
  <tr>
   <td>5011
   </td>
   <td>OpFragmentMaskFetchAMD
   </td>
  </tr>
  <tr>
   <td>5012
   </td>
   <td>OpFragmentFetchAMD
   </td>
  </tr>
  <tr>
   <td>5283
   </td>
   <td>OpImageSampleFootprintNV
   </td>
  </tr>
  <tr>
   <td>5296
   </td>
   <td>OpGroupNonUniformPartitionNV
   </td>
  </tr>
  <tr>
   <td>5299
   </td>
   <td>OpWritePackedPrimitiveIndices4x8NV
   </td>
  </tr>
  <tr>
   <td>5334
   </td>
   <td>OpReportIntersectionNV
   </td>
  </tr>
  <tr>
   <td>5335
   </td>
   <td>OpIgnoreIntersectionNV
   </td>
  </tr>
  <tr>
   <td>5336
   </td>
   <td>OpTerminateRayNV
   </td>
  </tr>
  <tr>
   <td>5337
   </td>
   <td>OpTraceNV
   </td>
  </tr>
  <tr>
   <td>5341
   </td>
   <td>OpTypeAccelerationStructureNV
   </td>
  </tr>
  <tr>
   <td>5344
   </td>
   <td>OpExecuteCallableNV
   </td>
  </tr>
  <tr>
   <td>5571
   </td>
   <td>OpSubgroupShuffleINTEL
   </td>
  </tr>
  <tr>
   <td>5572
   </td>
   <td>OpSubgroupShuffleDownINTEL
   </td>
  </tr>
  <tr>
   <td>5573
   </td>
   <td>OpSubgroupShuffleUpINTEL
   </td>
  </tr>
  <tr>
   <td>5574
   </td>
   <td>OpSubgroupShuffleXorINTEL
   </td>
  </tr>
  <tr>
   <td>5575
   </td>
   <td>OpSubgroupBlockReadINTEL
   </td>
  </tr>
  <tr>
   <td>5576
   </td>
   <td>OpSubgroupBlockWriteINTEL
   </td>
  </tr>
  <tr>
   <td>5577
   </td>
   <td>OpSubgroupImageBlockReadINTEL
   </td>
  </tr>
  <tr>
   <td>5578
   </td>
   <td>OpSubgroupImageBlockWriteINTEL
   </td>
  </tr>
  <tr>
   <td>5632
   </td>
   <td>OpDecorateStringGOOGLE
   </td>
  </tr>
  <tr>
   <td>5633
   </td>
   <td>OpMemberDecorateStringGOOGLE
   </td>
  </tr>
</Table>
