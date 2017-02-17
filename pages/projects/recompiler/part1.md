---
layout: project_page
title: Recompiler - CPU
permalink: /projects/recompiler/cpu/
project_root: recompiler
---

## XEX

First, the XEX (Xbox Executable) format must be ripped apart and the actual code has to be extracted. XEX is a Xbox360 specific executable packing/encryption format. It's not very complicated and quite good description can be found here: [Free60](http://free60.org/wiki/Main_Page). There are also some old references [here](http://www.openrce.org/forums/posts/111). Inside the XEX there are some platform specific headers (like file certificate, media/region information, file encryption key, etc) but also there's a normal 
PE style executable, unfortunatelly it's packed and encrypted.

Decryption of any actual executables requires knowing the secret AES key that is used internally by the loader to compute another AES key that is actually used to decrypt the file content. I found it on a Russian site few years ago but could not retrace my steps any more, most likely the site is down gone for good. The rest of the XEX format suggests strongly that it was bascically built on top of existing PE image loader that existed in the OS. The compression used in the XEX is either simple block based compression or a variation of LZ compression. Both were identified and reversed years ago by people trying to break the Xbox360 anti-piracy protection.

Any way, after dealing with those two bumps on the road and unpacking the "internal EXE" from the XEX we follow normal disassembly procedure. I general case we end up with a list of sections:

```
.rdata   0x00000400-0x0004C100 r__
.pdata   0x0004C200-0x00053358 r__
.text    0x00060000-0x0026B158 r_x
.data    0x00270000-0x00289D50 rw_
.XBMOVIE 0x00289E00-0x00289E08 rw_
.idata   0x00290000-0x00290268 rw_
.XBLD    0x002A0000-0x002A0090 r__
.reloc   0x002A0200-0x002B2EF4 r__
```

The only section that contains executable code is the .text section and only that section requires disassembly. Rest of the sections must still be loaded into the memory when we try to execute the code since data may be read/written into those addresses. For now I did not implement any data relocation so the unpacked image must be loaded exactly under it's base address. Fortunatelly on 64-bit systems this is achievable fairly easy.

## Disassembly

Disassembling the PowerPC instructions is a pleasure. After we identify the .text (code) section in the PE executable the rest is straightforward. Every instruction is always 4 bytes so there is no ambigiuity like with x86 instructions and even if we don't know how to decode a particular instruction we can easily continue with the rest of the code. This allows us to have semi-working solution way sooner than with x86. See the [actual file](https://github.com/rexdex/recompiler/blob/master/dev/src/platforms/xenon/xenonDecompiler/xenonInstructionDecoding.cpp) for details. 

Basically in this project I've tried 3 ways to approach the subject:

- Script based (for faster iteration) - I've written a LUA script that was doing the disasembly. It was fast to iterate in small samples but very slow to run and disassemble milions of instructions in normal executables was taking minutes. Even to check if an instruction is valid instruction was taking way to much time.

- Data drive pattern matching - Basically an big XML with binary "rules" that were mattching bit patterns and emitting instructions. This was much faster but because of the corner cases in the PowerPC instruction encoding it got messy in the end and required me to do a lot of copy-pasting. Performance wise it was fast and could work if not for a one little detail: it's not enough just to disassemble the code, we still need to extract some "meta data" out of the code (like register dependnecies, calculated jump addresses, etc). This still requires us to know a little bit about the instruction that just it's name. So, the template-based diassembler was producing an instruction named "bc" but I stil had to write manual code to understand that it's a "conditional branch" and even more code to be able to evaluate this condition.

- Finally I ended up with an abstract CPUInstruction class that is implemented for every instruction that CPU implements + a big ass C++ switch() to do the disassemblly. This is actually very nice and maintaintable solution.

The ***biggest*** and most valuable resources on this topic were the official PowerPC instruction set documentation: [Power ISA Version 2.07](http://fileadmin.cs.lth.se/cs/education/EDAN25/PowerISA_V2.07_PUBLIC.pdf)

## Testing the disassembler

I had lots of bugs in the disassembler. Of course I could write an unit test for each instruction but that would just take ages. The fastest way I've found to test the corectness of the disassembly is to compare the output with something that we know works. Basically, disassembling any big PowerPC executable by IDA or any other disassemble and comparing the output of tens of milions of instructions is a very good step towards some level of trust that the disassembler is working :)

## XBox360 specific instructions

Xbox360 has a special version of the PowerPC CPU that has 128 VMX registers (instead of 32 ones in the standard CPU). Those registers are used for vectorized math operations (similary to SSE). There's no way to address 128 registers in normal PowerPC instructions because there are only 5 bits delgated to indicate the register index in every instruction and this pattern is CPU-wide. Unfortunatelly, the opcodes for those special instructions are not avaiable on the internet (or are burried deeply). I ended up reversing the opcodes manually by observing the bit patters in the generated listing files. There's a simple tool I've wrote for that [XPrint](https://github.com/rexdex/recompiler/blob/master/dev/tools/xprint/XOpPrint.cpp). Typical output of a decoded instruction bit pattern looks like that:

```
Instruction 'vxor128' (3 params)
 Mask: '000101** ******** ******11 00*1****'
  Field[0]: 27:1 (1)
  Field[1]: 22:4 (12)
  Field[2]: 0:6 (5)
    Variants for param 0 (10):
        Mask:            ------** ***----- -------- ----**--
        Variant    vr0:  ------00 000***** *****0-- --0-0000
        Variant    vr1:  ------00 0010000* 0000**-- --*-00**
		....
        Variant   vr97:  ------00 00100001 00001*-- --*-11**
    Variants for param 1 (10):
        Mask:            -------- ---***** -----*-- --*-----
        Variant    vr0:  ------** ***00000 *****0-- --0-0000
        Variant    vr1:  ------00 00*00001 0000*0-- --0-****
        Variant    vr2:  ------00 00000010 000000-- --0-0000
		....
        Variant   vr97:  ------00 00100001 000011-- --1-****
    Variants for param 2 (10):
        Mask:            -------- -------- *****--- ------**
        Variant    vr0:  ------** ******** 000000-- --0-0000
        Variant    vr1:  ------00 00*0000* 00001*-- --*-**00
        Variant    vr2:  ------00 00000000 000100-- --0-0000
		....
        Variant   vr97:  ------00 00100001 00001*-- --*-**11
```

## Abstract instruction

The result of disassembly process is an "unpacked instruction".  he most useful thing is that the opcode and operands are unpacked so an easy "ToString" method can be written for the presentation purposes. Surprisingly, this structure captures a lot of quirks of not only the PowerPC instructions but Intel as well. On PowerPC the operand type is closely related to the particular instruction type (add vs addi, etc). On Intel this is not the case and the same instruction (add) may be used with immediate value as well as memory location, etc. To capture this generalization the Operand structure is introduced.

```c++
class  Instruction
{
public:
	// register reference (uses the CPU definition directly)
	typedef const platform::CPURegister* TReg;

	// opcode reference
	typedef const class platform::CPUInstruction* TOpcode;
		
	// operand addressing mode
	enum Type
	{
		eType_None,	// invalid (not defined)
		eType_Reg,	// register only, "r"
		eType_Imm,	// immediate only "i"
		eType_Mem,	// dereferenced register "[r] or [r+d+idx*scale]"
	};

	// instruction operand 
	struct Operand
	{
		Type    m_type;    // common
		uint8   m_scale;   // x86 only
		TReg    m_reg;     // common
		uint32  m_imm;     // immediate value, always SIGN EXTENDED if needed
		TReg    m_index;   // index register
		TReg    m_segment; // segment register

		inline Operand()
			: m_type( eType_None )
			, m_reg( nullptr )
			, m_scale( 1 )
			, m_imm( 0 )
			, m_index( nullptr )
			, m_segment( nullptr )
		{}
	};

protected:
	// size of the instruction (in bytes)
	uint8 m_codeSize;

	// size of used data in the instruction - 2, 4, 8
	uint8 m_dataSize;

	// size of address calculation registers, 2, 4, 8
	uint8 m_addrSize;

	// instruction opcode
	TOpcode m_opcode;

	// operands (not all instructions use all)
	Operand m_arg0;
	Operand m_arg1;
	Operand m_arg2;
	Operand m_arg3;
	Operand m_arg4;
	Operand m_arg5;
};
```

In practice, the unpacked format is not good enough for many operations. After the diassembly is completed more work is needed to get the code to a useful state than just unpacking. For example we need to identify the "blocks" - a places in the code where execution enters a particular linear set of instruction that are going to be executed without interuptions until a "jump" or "call" to another block. It's nice and usefull to abstract this instruction concept a little bit more. This is done by the following class:

```c++
class InstructionExtendedInfo
{
public:
	static const uint32 MAX_REGISTERS = 8;

	enum EMemoryFlags
	{
		eMemoryFlags_Read		= 1 << 0, // memory is being read by the instruction
		eMemoryFlags_Write		= 1 << 1, // memory is being written by the instruction
		eMemoryFlags_Touch		= 1 << 2, // memory is being touched but not read/write (cache)
		eMemoryFlags_Depends	= 1 << 3, // instruction depends on the memory content in some obscure way
		eMemoryFlags_Aligned	= 1 << 4, // instruction expects memory address to be aligned
		eMemoryFlags_Float		= 1 << 5, // memory is expected to be loading point value
		eMemoryFlags_DirectMap  = 1 << 6, // memory access is directly mapped (write/read instructions)
	};

	enum EInstructionFlags
	{
		eInstructionFlag_Nop		= 1 << 0, // instruction can be removed
		eInstructionFlag_Jump		= 1 << 1, // instruction represents a simple jump
		eInstructionFlag_Call		= 1 << 2, // instruction represents a function call
		eInstructionFlag_Return		= 1 << 3, // instruction represents a return from function
		eInstructionFlag_Diverge	= 1 << 4, // instruction can diverge a flow of code (conditional branch)
		eInstructionFlag_Conditional= 1 << 5, // instruction is conditional (uses condition registers or flags)
		eInstructionFlag_ModifyFlags= 1 << 6, // instruction is modifying the flag registers
		eInstructionFlag_Static		= 1 << 7, // instruction branch target is static
	};

	enum ERegDependencyMode
	{
		eReg_None = 0,
		eReg_Dependency = 1,
		eReg_Output = 2,
		eReg_Both = 3,
	};

	// instruction address and flags
	uint32 m_codeAddress;
	uint32 m_codeFlags;

	// speculated branch target (if known)
	uint64 m_branchTargetAddress;
	const platform::CPURegister* m_branchTargetReg;

	// memory access info (if any)
	uint64 m_memoryAddressOffset;
	const platform::CPURegister* m_memoryAddressBase;
	const platform::CPURegister* m_memoryAddressIndex;
	uint32 m_memoryAddressScale;
	uint32 m_memoryFlags;
	uint32 m_memorySize;

	// registers read (a fully modified register is NOT a dependency)
	const platform::CPURegister* m_registersDependencies[ MAX_REGISTERS ];
	uint32 m_registersDependenciesCount;

	// registers modified
	const platform::CPURegister* m_registersModified[ MAX_REGISTERS ];
	uint32 m_registersModifiedCount;
};
```

By filling in this class the decompiler can express much more about the instruction - ie. what is it going to do with the control flow of the program or what registeres are being read/written by it. Each actual PowerPC instruction has it's opcode class that is able to transform the Instruction into the InstructionExtendedInfo.

## Blocks

After all instructions are disassembled it's important to identify blocks of instructions that can have known single place of entry. This is done by analyzing all the "call" and "jump" instructions that can be resolved. This is not foolproof as it does not identify properly the indirect calls (vtable, function pointers) and indirect jumps (switch statements). The more knowledge about a block we have and the more certnity about the points of entry the faster code we will be able to generate.

![DolphinDemoScreenshot](/images/projects/xbox_blocks.jpg)

## Recompilation

After all of the code is diassembled we can start to decompile it into logically equivalent representation. The simple trick here is to realize that for the sake of just getting it to work we don't need to convert the code into any high-level language. What matters is to get exactly the same execution results. The CPU state is represented as a structure:

```c++
class CpuRegs : public runtime::RegisterBank
{
public:
	TReg		LR;
	TReg		CTR;
	TReg		MSR;
	XerReg		XER;

	ControlReg	CR[8];

	uint32		FPSCR;

	TReg		R0;
	TReg		R1;
	/* ... */
	TReg		R32;

	TFReg		FR0;
	TFReg		FR1;
	/* ... */
	TFReg		FR31;

	VReg		VR0;
	VReg		VR1;
	/* ... */
	VReg		VR127;

	Reservation* RES;	
};
```

All the PowerPC instructions are rewritten as a heavely templatized and inlined C++ functions:

```c++
// addic - add immediate with the update of the carry flag
template <uint8 CTRL>
static inline void addic( CpuRegs& regs, TReg* out, const TReg a, const uint32 imm )
{
	*out = a + EXTS(imm);
	regs.XER.ca = (*out < a); // carry assuming there was no carry before
	if (CTRL) cmp::CmpSignedXER<0>(regs, *(int64*)out, 0);
}
```

Finally all the blocks that were identified are transformed 1-1 into equlivalent block functions. Blocks function signature is following:

```c++
uint64 __fastcall _code__block82060508( uint64 ip, cpu::CpuRegs& regs )
```

It takes the current IP (instruction pointer) directly as the argument + the current CPU state expressed by the "regs". The returned value represents next address to execute. This is the lowest (slowest) code generation level that we have. In this mode we are putting ***ALL BURDEN*** of optimizing this to final assembly onto the target compiler. Supprisingly, even using this naive approach most of the recompiled executables are running supprisingly well. Typical block looks like that:

```c++
//////////////////////////////////////////////////////
// Block at 82060508h
//////////////////////////////////////////////////////
uint64 __fastcall _code__block82060508( uint64 ip, cpu::CpuRegs& regs )
{
	const uint32 local_instr = (uint32)(ip - 0x82060508) / 4;
	switch ( local_instr )
	{
		default:	cpu::invalid_address( ip, 0x82060508 );
		case 0: cpu::op::lis<0>(regs,&regs.R11,0xFFFF8229);
		case 1: cpu::mem::load32z( regs, &regs.R10, (uint32)(regs.R1 + 0x000000A0) );
		case 2: cpu::op::lis<0>(regs,&regs.R9,0xFFFF8229);
		case 3: cpu::op::addi<0>(regs,&regs.R3,regs.R1,0xA0);
		case 4: cpu::mem::load32z( regs, &regs.R11, (uint32)(regs.R11 + 0xFFFF8BA8) );
		case 5: cpu::mem::load32z( regs, &regs.R10, (uint32)(regs.R10 + 0x00000000) );
		case 6: cpu::mem::store32( regs, regs.R11, (uint32)(regs.R9 + 0xFFFF8BA4) );
		case 7: regs.CTR = regs.R10;
		case 8: if ( 1 ) { regs.LR = 0x8206052C; return (uint32)regs.CTR; }
		case 9: cpu::op::cmpwi<0>(regs,regs.R3,0x00000000);
		case 10: if ( !regs.CR[0].lt ) { return 0x82060554;  }
		case 11: regs.R4 = regs.R30;
		case 12: regs.R3 = regs.R31;
		case 13: cpu::op::li<0>(regs,&regs.R5,0x50);
		case 14: regs.LR = 0x82060544; return 0x820859E0;
		case 15: cpu::op::lis<0>(regs,&regs.R11,0xFFFF8205);
		case 16: cpu::op::addi<0>(regs,&regs.R3,regs.R11,0xFFFFA98C);
		case 17: regs.LR = 0x82060550; return 0x820859E0;
		case 18: regs.LR = 0x82060554; return 0x82267AF8;
	}
	return 0x82060554;
}
```

In case the block is confirmed to be single point of entry we can use following simplified form:

```c++
//////////////////////////////////////////////////////
// Block at 82060508h
//////////////////////////////////////////////////////
uint64 __fastcall _code__block82060508( uint64 ip, cpu::CpuRegs& regs )
{
	ASSERT(ip == 0x82060508);
	cpu::op::lis<0>(regs,&regs.R11,0xFFFF8229);
	cpu::mem::load32z( regs, &regs.R10, (uint32)(regs.R1 + 0x000000A0) );
	cpu::op::lis<0>(regs,&regs.R9,0xFFFF8229);
	cpu::op::addi<0>(regs,&regs.R3,regs.R1,0xA0);
	cpu::mem::load32z( regs, &regs.R11, (uint32)(regs.R11 + 0xFFFF8BA8) );
	cpu::mem::load32z( regs, &regs.R10, (uint32)(regs.R10 + 0x00000000) );
	cpu::mem::store32( regs, regs.R11, (uint32)(regs.R9 + 0xFFFF8BA4) );
	regs.CTR = regs.R10;
	if ( 1 ) { regs.LR = 0x8206052C; return (uint32)regs.CTR; }
	cpu::op::cmpwi<0>(regs,regs.R3,0x00000000);
	if ( !regs.CR[0].lt ) { return 0x82060554;  }
	regs.R4 = regs.R30;
	regs.R3 = regs.R31;
	cpu::op::li<0>(regs,&regs.R5,0x50);
	regs.LR = 0x82060544; return 0x820859E0;
	cpu::op::lis<0>(regs,&regs.R11,0xFFFF8205);
	cpu::op::addi<0>(regs,&regs.R3,regs.R11,0xFFFFA98C);
	regs.LR = 0x82060550; return 0x820859E0;
	regs.LR = 0x82060554; return 0x82267AF8;
	return 0x82060554;
}
```

There are more optimization steps possible that I'm currently working on - for example if all blocks in a function are "well behaved" - no indirect jumps are found and the function follows ABI rules - clear preambule can be identified + there are all return statements have proper cleanup code than we can promote the whole function to a single C++ function pulling all blocks inside + defining all VOLATILE registers inside the function (on stack) and not using the ones in the *regs* structure. 

Next optimization step can occur when two "well behaved" functions are calling each other. Then, instead of going through the generic call via the returned "next instruction address" we can generate code like this:

```c++
uint64 __fastcall _code__func82060508( uint64 ip, cpu::CpuRegs& regs )
{
	ASSERT(ip == 0x82060508);
	
	/*...*/
	
	// the CALL_DIRECT checks if the returned address is what we expected 
	CALL_DIRECT(0x82060608, _code__func82010004(0x82010004, regs)); 

	/*...*/
	
	return 0x82060554;
}
```

This again makes the generated code faster.

## The thread and the thread execution.

All the generated blocks are than compiled using standard C++ compiler and produce a DLL. Pointers to all block functions are then registered into a "block table". Block table allows easily to retrieve the block that will contain the code for given IP (Instruction Pointer). Finally the the core of the simulated CPU thread boils down to this function:

```c++
void CodeExecutor::Run()
{
	while (m_ip)
	{
		const auto relAddress = m_ip - m_startAddress;
		const auto func = m_blockTable[relAddress];
		m_ip = func(m_ip, *m_regs);
	}
}
```

## Imported functions

The XEX image contains import of functions from another modules. Unlink the quite common "named" imports, the ones in the XEX executable are only ordinal based. A table is required that contans the "human readable" names of the functions as well as their ordering in the given lib. See [here](https://github.com/rexdex/recompiler/blob/master/dev/src/platforms/xenon/xenonDecompiler/Recompiler.Xenon.Platform.exports).

When we load an image for decompiled executable we can patch the entries in the block table for given import stubs with a C++ reimplementation of that function. The is still the same and we have to "unpack" the arguments from the registers manually. For example:

```c++
uint64 __fastcall XboxThreads_KeDelayExecutionThread( uint64 ip, cpu::CpuRegs& regs )
{
	const uint32 processor_mode = (const uint32)regs.R3;
	const uint32 alertable = (const uint32)regs.R4;
	const uint32 internalPtr = (const uint32)regs.R5;
	const uint64 interval = mem::loadAddr<uint64>( internalPtr );

	auto* currentThread = xenon::KernelThread::GetCurrentThread();
	const uint32 result = currentThread->Delay(processor_mode, alertable, interval);

	RETURN_ARG(result);
}
```

It takes around 300 functions to get the simple app to start. Most of the are very similar (if not exactly the same) as the Windows counterparts. The rest is mostly guess work.


