# WebGPU Execution Environment

**Note** This document is meant to be a reference for understanding and discussing WebGPU & WGSL. It is not meant to an implemented SPIR-V environment.
**Note** This document is in transition.
It started as a specification of the form and semantics SPIR-V modules that could be _directly ingestible_ as the shader language for WebGPU.
It is transitioning to a specification of the form and semantics of SPIR-V modules that should be _translatable into WGSL_, such that
the corresponding WGSL shaders are ingestible as shaders in WebGPU.

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

WebGPU shaders are expressed in WGSL.
However, a SPIR-V module can often be translated into WGSL for use in WebGPU.
SPIR-V is usable in several execution environments, and so its core definition describes a superset of functionality across all those environments.
The "WebGPU Execution Environment Specification for SPIR-V" section describes restrictions on SPIR-V such that they can be expected
to be translated into WGSL for use with WebGPU.

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

Shaders in SPIR-V that may be translated into WGSL are specified by this rules in this document in addition to the [Khronos SPIR-V Specification](https://www.khronos.org/registry/spir-v/specs/unified1/SPIRV.html).
[TBD: update to latest available specification] See the "Version" subsection below for allowed version numbers.

The following sections visit the parts of a SPIR-V module, in order, covering each section, and state additional requirements for SPIR-V shaders translatable for use in WebGPU. These are in addition to all requirements already present in the SPIR-V specification itself.

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

SPIR-V versions 1.0, 1.1, 1.2, 1.3, 1.4, 1.5 should be usable: The version number (second 32-bit word) of a SPIR-V module must be one of the following:

*   0x00010000 (SPIR-V 1.0)
*   0x00010100 (SPIR-V 1.1)
*   0x00010200 (SPIR-V 1.2)
*   0x00010300 (SPIR-V 1.3)
*   0x00010400 (SPIR-V 1.4)
*   0x00010500 (SPIR-V 1.5)

Features introduced in newer versions might still be unavailable due to restrictions introduced later in this specification.

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
*   SPV_KHR_storage_buffer_storage_class
*   SPV_KHR_no_integer_wrap_decoration
*   SPV_KHR_non_semantic_info
*   SPV_GOOGLE_decorate_string
*   SPV_GOOGLE_hlsl_functionality1
*   SPV_GOOGLE_user_type
*   [No other extensions allowed for now]


### Extended Instruction Sets

All extended instruction sets declared by **OpExtInstImport** must be one of the following:


*   "GLSL.std.450"
*   A non-semantic extended instruction set following the conventions in
    [SPV_KHR_non_semantic_info](https://htmlpreview.github.io/?https://github.com/KhronosGroup/SPIRV-Registry/blob/master/extensions/KHR/SPV_KHR_non_semantic_info.html)
    In particular, "NonSemantic." is a prefix of the extended instruction set name.


### SPIR-V Memory Model

The addressing declared by **OpMemoryModel** must be **Logical**.

The memory model declared must be one of:

* **Simple**
* **GLSL450**
* **Vulkan**, also known as **VulkanKHR**


### Entry Points

There must be at least one **OpEntryPoint** in a module, and each one present must satisfy:



* The **OpFunction** declared as the entry point must have no **OpFunctionParameter** and  its _Return Type_ must be **OpTypeVoid**.
* The complete call graph rooted by the entry point must
  *   be present in the module (the **Linkage** capability is not valid, so there are no partially linked modules), and
  *   not contain any cycles (no recursion, even statically).
* The _Name_ of an **OpEntryPoint**
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

[//]: # (No subgroups support. Useful, but not widely available?)



### Debug Instructions

The following debug instructions are permitted: OpName, OpMemberName, OpLine, OpNoLine, OpModuleProcessed, OpString, OpSourceExtension, OpSource, and OpSourceContinued.


### Annotations

OpDecorationGroup, OpGroupDecorate, OpGroupMemberDecorate are not allowed.

Only the following decorations are valid:



[//]: # (No **RelaxedPrecision**: WHLSL doesn't have min16float min10float)
[//]: # (No **Sample**: WebGPU does not support sample rate shading)
[//]: # (No **Invariant**: Assume WebGPU MVP does not support this for complexity reasons)
[//]: # (No **Coherent**: Assume Vulkan memory model)

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
*   **NoSignedWrap**
*   **NoUnsignedWrap**
*   **CounterBuffer**, also known as **HlslCounterBufferGOOGLE**
*   **UserSemantic**, also known as **HlslSemanticGOOGLE**
*   **UserTypeGOOGLE**
*   **UniformId**


#### Built-In Variables

When decorating with the **Builtin** decoration, only the following are valid, and these are further restricted as indicated by the table.


Builtin | Allowed types | Allowed execution models for the Input storage class | Allowed execution models for the Output storage class
--------|---------------|------------------------------------------------------|------------------------------------------------------
**Position** | Vector of 4 32-bit floats |  | **Vertex**
**VertexIndex** | Scalar 32-bit integer | **Vertex** |
**InstanceIndex** | Scalar 32-bit integer | **Vertex** |
**FrontFacing** | Boolean | **Fragment** |
**FragCoord** | Vector of 4 32-bit floats | **Fragment** |
**FragDepth** | 32-bit float |  | **Fragment**
**WorkgroupSize** | Vector of 3 32-bit ints | **GLCompute** |
**LocalInvocationId** | Vector of 3 32-bit ints | **GLCompute** |
**GlobalInvocationId** | Vector of 3 32-bit ints | **GLCompute** |
**LocalInvocationIndex** | 32-bit int | **GLCompute** |

Note: **NumWorkgroups** is not in the WebGPU MVP. See https://github.com/gpuweb/gpuweb/issues/920

Full semantic descriptions of what these mean, how the behave, when they have what values, etc. is part of the WebGPU specification proper, not this specification.


### Types

Scalar floating-point types (**OpTypeFloat**) must have 32 bit width.

Scalar integer types (**OpTypeInteger**) must have 32 bit width.

Signed integers use two's complement representation.

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

All functions in the SPIR-V module must use structured control flow.
See [2.11 Structured Control Flow](https://www.khronos.org/registry/spir-v/specs/unified1/SPIRV.html#StructuredControlFlow).

#### Scope

**Scope** when used for memory must be one of:

[//]: # (No **Device** since that's optional for Vulkan memory model.  In single-device configurations QueueFamilyKHR is the same as Device)

*   **Workgroup**
*   **Invocation**
*   **QueueFamilyKHR**

**Workgroup** memory scope is only usable in a **GLCompute** shader.

**Scope** when used for execution must be one of:

*   **Workgroup**

**Workgroup** execution scope is only usable in a **GLCompute** shader.

[//]: # (Vulkan limits execution scope to Workgroup and Subgroup, but WebGPU MVP does not have subgroup ops)

Individual SPIR-V instructions may further restrict valid values.

#### Memory Semantics

A Memory Semantics operand is expressed in a single word,
using mask bits in grouped into three sets:

*   _Memory Semantics Order bits_ are mask bits with value between 1 through 0x10:
    * 0x1 _Reserved_
    * 0x2 **Acquire**
    * 0x4 **Release**
    * 0x8 **AcquireRelease**
    * 0x10 **SequentiallyConsistent**
*   _Memory Semantics Storage Class bits_ are mask bits with value between 0x20 through 0x1000:
    * 0x20 _Reserved_
    * 0x40 **UniformMemory**
    * 0x80 **SubgroupMemory**
    * 0x100 **WorkgroupMemory**
    * 0x200 **CrossWorkgroupMemory**
    * 0x400 **AtomicCounterMemory**
    * 0x800 **ImageMemory**
    * 0x1000 **OutputMemoryKHR**
*   _Memory Semantics Propagation bits_ are mask bits with value between 0x2000 through 0x4000:
    * 0x2000 **MakeAvailableKHR**
    * 0x4000 **MakeVisibleKHR**

Individual SPIR-V instructions may further restrict valid values for a _Memory Semantics
operand_.
The restrictions are expressed in terms of the order, storage class, and propagation
sets of bits.

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

Supported OpCodes are listed in [Appendix A](#a-supported-opcodes).

The following sections describe differences from specific SPIR-V instructions.

####  Miscellaneous instructions

*   **OpUndef** is not allowed.

####  Constant creation instructions

*   **OpSpecConstantOp**:
  * Opcode **OpVectorShuffle** may not have a component literal with value 0xFFFFFFFF.

####  Memory instructions

* **Workgroup** variables are only usable in **GLCompute** shaders.
  Specifically:
  * If instruction _Inst_ is in the function parameter list or the body of
    a function in the call graph of an entry point _E_, and
  * The type of any operand of _Inst_ or the result of _Inst_ is a pointer type with
    storage class **Workgroup**,
  * Then _E_ must have an execution model of **GLCompute**.

* **OpPtrEqual**, **OpPtrNotEqual**, **OpPtrDiff** are not allowed.

####  Composite instructions

*   OpVectorShuffle may not have a component literal with value 0xFFFFFFFF.

####  Atomic instructions

*  Atomic instructions (**OpAtomic**\*) are restricted:
    *   The _Memory_ scope operand must be **QueueFamilyKHR**
    *   The _Semantics_ operand must be zero:
        * Must not set any Memory Semantics Order bits.
        * Must not set any Memory Semantics Storage Class bits.
        * Must not set any Memory Semantics Propagation bits.

Note: In terms from other standards, **OpAtomic**\* instructions are _Relaxed_.

Note: In the memory model, atomic operations automatically include
availability and visibility semantics.

####  Barrier instructions


*  **OpControlBarrier**\* restrictions:
    * **OpControlBarrier** may only be used in the call graph of **GLCompute**
    entry point.
    *   The _Execution_ scope operand must be **Workgroup**
    *   The _Memory_ scope operand must be **Workgroup**
    *   The _Semantics_ operand:
        * Memory Semantics Order bits: Must set the **AcquireRelease** bit, and no other bits.
        * Memory Semantics Storage Class bits: Must set the **WorkgroupMemory** bit, and no other bits.
        * Must not set any Memory Semantics Propagation bits.

Note: To ensure propagation of **Workgroup** writes to readers,
individual accesses of **Workgroup** memory should use availability
and visibility semantics:
**MakePointerAvailableKHR** with **Workgroup** scope on writes,
**MakePointerVisibleKHR** with **Workgroup** scope on reads,
and **NonPrivatePointerKHR** on both.

*  **OpMemoryBarrier**\* restrictions:
    *   The _Memory_  scope operand must be **Invocation**
    *   The _Semantics_ operand:
        * Must not set any Memory Semantics Order bits.
        * Memory Semantics Storage Class bits: Must set the **ImageMemory** bit, and no other bits.
        * Must not set any Memory Semantics Propagation bits.

Note: **OpMemoryBarrier** is only used to order reads and writes by the same
invocation to the same locations in image memory.

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
Scalars are represented in little-endian byte order.

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

OpCode | Name
-------|-----
0 | OpNop
10 | OpExtension
11 | OpExtInstImport
12 | OpExtInst
14 | OpMemoryModel
15 | OpEntryPoint
16 | OpExecutionMode
17 | OpCapability
19 | OpTypeVoid
20 | OpTypeBool
21 | OpTypeInt
22 | OpTypeFloat
23 | OpTypeVector
24 | OpTypeMatrix
25 | OpTypeImage
26 | OpTypeSampler
27 | OpTypeSampledImage
28 | OpTypeArray
29 | OpTypeRuntimeArray
30 | OpTypeStruct
32 | OpTypePointer
33 | OpTypeFunction
41 | OpConstantTrue
42 | OpConstantFalse
43 | OpConstant
44 | OpConstantComposite
46 | OpConstantNull
48 | OpSpecConstantTrue
49 | OpSpecConstantFalse
50 | OpSpecConstant
51 | OpSpecConstantComposite
52 | OpSpecConstantOp
54 | OpFunction
55 | OpFunctionParameter
56 | OpFunctionEnd
57 | OpFunctionCall
59 | OpVariable
60 | OpImageTexelPointer
61 | OpLoad
62 | OpStore
63 | OpCopyMemory
65 | OpAccessChain
66 | OpInBoundsAccessChain
68 | OpArrayLength
71 | OpDecorate
72 | OpMemberDecorate
77 | OpVectorExtractDynamic
78 | OpVectorInsertDynamic
79 | OpVectorShuffle
80 | OpCompositeConstruct
81 | OpCompositeExtract
82 | OpCompositeInsert
83 | OpCopyObject
84 | OpTranspose
86 | OpSampledImage
87 | OpImageSampleImplicitLod
88 | OpImageSampleExplicitLod
89 | OpImageSampleDrefImplicitLod
90 | OpImageSampleDrefExplicitLod
91 | OpImageSampleProjImplicitLod
92 | OpImageSampleProjExplicitLod
93 | OpImageSampleProjDrefImplicitLod
94 | OpImageSampleProjDrefExplicitLod
95 | OpImageFetch
96 | OpImageGather
97 | OpImageDrefGather
98 | OpImageRead
99 | OpImageWrite
100 | OpImage
103 | OpImageQuerySizeLod
104 | OpImageQuerySize
105 | OpImageQueryLod
106 | OpImageQueryLevels
107 | OpImageQuerySamples
109 | OpConvertFToU
110 | OpConvertFToS
111 | OpConvertSToF
112 | OpConvertUToF
113 | OpUConvert
114 | OpSConvert
115 | OpFConvert
116 | OpQuantizeToF16
124 | OpBitcast
126 | OpSNegate
127 | OpFNegate
128 | OpIAdd
129 | OpFAdd
130 | OpISub
131 | OpFSub
132 | OpIMul
133 | OpFMul
134 | OpUDiv
135 | OpSDiv
136 | OpFDiv
137 | OpUMod
138 | OpSRem
139 | OpSMod
140 | OpFRem
141 | OpFMod
142 | OpVectorTimesScalar
143 | OpMatrixTimesScalar
144 | OpVectorTimesMatrix
145 | OpMatrixTimesVector
146 | OpMatrixTimesMatrix
147 | OpOuterProduct
148 | OpDot
149 | OpIAddCarry
150 | OpISubBorrow
151 | OpUMulExtended
152 | OpSMulExtended
154 | OpAny
155 | OpAll
156 | OpIsNan
157 | OpIsInf
164 | OpLogicalEqual
165 | OpLogicalNotEqual
166 | OpLogicalOr
167 | OpLogicalAnd
168 | OpLogicalNot
169 | OpSelect
170 | OpIEqual
171 | OpINotEqual
172 | OpUGreaterThan
173 | OpSGreaterThan
174 | OpUGreaterThanEqual
175 | OpSGreaterThanEqual
176 | OpULessThan
177 | OpSLessThan
178 | OpULessThanEqual
179 | OpSLessThanEqual
180 | OpFOrdEqual
181 | OpFUnordEqual
182 | OpFOrdNotEqual
183 | OpFUnordNotEqual
184 | OpFOrdLessThan
185 | OpFUnordLessThan
186 | OpFOrdGreaterThan
187 | OpFUnordGreaterThan
188 | OpFOrdLessThanEqual
189 | OpFUnordLessThanEqual
190 | OpFOrdGreaterThanEqual
191 | OpFUnordGreaterThanEqual
194 | OpShiftRightLogical
195 | OpShiftRightArithmetic
196 | OpShiftLeftLogical
197 | OpBitwiseOr
198 | OpBitwiseXor
199 | OpBitwiseAnd
200 | OpNot
201 | OpBitFieldInsert
202 | OpBitFieldSExtract
203 | OpBitFieldUExtract
204 | OpBitReverse
205 | OpBitCount
207 | OpDPdx
208 | OpDPdy
209 | OpFwidth
210 | OpDPdxFine
211 | OpDPdyFine
212 | OpFwidthFine
213 | OpDPdxCoarse
214 | OpDPdyCoarse
215 | OpFwidthCoarse
224 | OpControlBarrier
225 | OpMemoryBarrier
227 | OpAtomicLoad
228 | OpAtomicStore
229 | OpAtomicExchange
230 | OpAtomicCompareExchange
232 | OpAtomicIIncrement
233 | OpAtomicIDecrement
234 | OpAtomicIAdd
235 | OpAtomicISub
236 | OpAtomicSMin
237 | OpAtomicUMin
238 | OpAtomicSMax
239 | OpAtomicUMax
240 | OpAtomicAnd
241 | OpAtomicOr
242 | OpAtomicXor
245 | OpPhi
246 | OpLoopMerge
247 | OpSelectionMerge
248 | OpLabel
249 | OpBranch
250 | OpBranchConditional
251 | OpSwitch
252 | OpKill
253 | OpReturn
254 | OpReturnValue
255 | OpUnreachable
331 | OpExecutionModeId
332 | OpDecorateId
333 | OpName
334 | OpMemberName
335 | OpLine
336 | OpNoLine
337 | OpModuleProcessed
338 | OpString
339 | OpSourceExtension
340 | OpSource
341 | OpSourceContinued
400 | OpCopyLogical
5632 | OpDecorateString
5633 | OpMemberDecorateString
