# Iced  [![Build status](https://ci.appveyor.com/api/projects/status/xldctwks3we9tcb4/branch/master?svg=true)](https://ci.appveyor.com/project/0xd4d/iced/branch/master)   [![NuGet](https://img.shields.io/nuget/v/Iced.svg)](https://www.nuget.org/packages/Iced/)


<img align="right" width="160px" height="160px" src="logo.png">

High performance x86 (16/32/64-bit) instruction decoder, encoder and formatter.
It can be used for static analysis of x86/x64 binaries, to rewrite code (eg. remove garbage instructions), to relocate code or as a disassembler.

- Supports all Intel and AMD instructions
- The decoder doesn't allocate any memory and is 2x-5x+ faster than other similar libraries written in C or C#
- Small decoded instructions, only 32 bytes
- The formatter supports masm, nasm, gas (AT&T) and Intel (xed) and there are many options to customize the output
- The encoder can be used to re-encode decoded instructions at any address
- The block encoder encodes a list of instructions and optimizes branches to short, near or 'long' (64-bit: 1 or more instructions)
- API to get instruction info, eg. read/written registers, memory and rflags bits; CPUID feature flag, flow control info, etc
- All instructions are tested (decode, encode, format, instruction info)

# Classes

See below for some examples. All classes are in the `Iced.Intel` namespace.

Decoder:

- `Decoder`
- `Instruction`
- `CodeReader`
	- `ByteArrayCodeReader`
- `ConstantOffsets`

Formatters:

- `Formatter`
	- `MasmFormatter`
	- `NasmFormatter`
	- `GasFormatter`
	- `IntelFormatter`
- `FormatterOptions`
	- `MasmFormatterOptions`
	- `NasmFormatterOptions`
	- `GasFormatterOptions`
	- `IntelFormatterOptions`
- `FormatterOutput`
	- `StringBuilderFormatterOutput`
- `SymbolResolver`

Encoder:

- `Encoder`
- `BlockEncoder`
- `CodeWriter`
- `ConstantOffsets`

Instruction info:

- `Instruction.GetInfo()`
- `InstructionInfo`
- `InstructionInfoFactory`
- `InstructionInfoExtensions`
- `MemorySizeExtensions`
- `RegisterExtensions`

# Examples

```C#
using System;
using System.Collections.Generic;
using Iced.Intel;

namespace Iced.Examples {
    static class Program {
        static void Main(string[] args) {
            DecoderFormatterExample();
            EncoderExample();
            InstructionInfoExample();
        }

        const int exampleCodeBitness = 64;
        const ulong exampleCodeRIP = 0x00007FFAC46ACDA4;
        const int exampleCodeNumInstructions = 13;
        static readonly byte[] exampleCode = new byte[] {
            0x48, 0x89, 0x5C, 0x24, 0x10, 0x48, 0x89, 0x74, 0x24, 0x18, 0x55, 0x57, 0x41, 0x56, 0x48, 0x8D,
            0xAC, 0x24, 0x00, 0xFF, 0xFF, 0xFF, 0x48, 0x81, 0xEC, 0x00, 0x02, 0x00, 0x00, 0x48, 0x8B, 0x05,
            0x18, 0x57, 0x0A, 0x00, 0x48, 0x33, 0xC4, 0x48, 0x89, 0x85, 0xF0, 0x00, 0x00, 0x00, 0x4C, 0x8B,
            0x05, 0x2F, 0x24, 0x0A, 0x00, 0x48, 0x8D, 0x05, 0x78, 0x7C, 0x04, 0x00, 0x33, 0xFF
        };

        /*
         * This method produces the following output:
00007FFAC46ACDA4 mov       [rsp+10h],rbx
00007FFAC46ACDA9 mov       [rsp+18h],rsi
00007FFAC46ACDAE push      rbp
00007FFAC46ACDAF push      rdi
00007FFAC46ACDB0 push      r14
00007FFAC46ACDB2 lea       rbp,[rsp-100h]
00007FFAC46ACDBA sub       rsp,200h
00007FFAC46ACDC1 mov       rax,[rel 7FFA`C475`24E0h]
00007FFAC46ACDC8 xor       rax,rsp
00007FFAC46ACDCB mov       [rbp+0F0h],rax
00007FFAC46ACDD2 mov       r8,[rel 7FFA`C474`F208h]
00007FFAC46ACDD9 lea       rax,[rel 7FFA`C46F`4A58h]
00007FFAC46ACDE0 xor       edi,edi
        */
        static void DecoderFormatterExample() {
            // You can also pass in a hex string, eg. "90 91 929394", or you can use your own CodeReader
            // reading data from a file or memory etc
            var codeReader = new ByteArrayCodeReader(exampleCode);
            var decoder = Decoder.Create(exampleCodeBitness, codeReader);
            decoder.InstructionPointer = exampleCodeRIP;
            ulong endRip = decoder.InstructionPointer + (uint)exampleCode.Length;

            // Formatters: Masm*, Nasm*, Gas* (AT&T) and Intel* (Intel XED)
            var formatter = new NasmFormatter();
            formatter.Options.AddDigitSeparators = true;
            formatter.Options.DigitSeparator = "`";
            formatter.Options.FirstOperandCharIndex = 10;
            var output = new StringBuilderFormatterOutput();
            while (decoder.InstructionPointer < endRip) {
                decoder.Decode(out var instr);
                // Don't use instr.ToString(), it allocates more, only shows masm syntax and you can't change any options
                formatter.Format(ref instr, output);
                Console.WriteLine($"{instr.IP64:X16} {output.ToStringAndReset()}");
            }
        }

        /*
         * This method produces the following output:
New code bytes:
0x48, 0x89, 0x5C, 0x24, 0x10, 0x48, 0x89, 0x74, 0x24, 0x18, 0x55, 0x57, 0x41, 0x56, 0x48, 0x8D
0xAC, 0x24, 0x00, 0xFF, 0xFF, 0xFF, 0x48, 0x81, 0xEC, 0x00, 0x02, 0x00, 0x00, 0x48, 0x8B, 0x05
0x18, 0x57, 0xEA, 0xFF, 0x48, 0x33, 0xC4, 0x48, 0x89, 0x85, 0xF0, 0x00, 0x00, 0x00, 0x4C, 0x8B
0x05, 0x2F, 0x24, 0xEA, 0xFF, 0x48, 0x8D, 0x05, 0x78, 0x7C, 0xE4, 0xFF, 0x33, 0xFF
Disassembled code:
00007FFAC48ACDA4 mov       [rsp+10h],rbx
00007FFAC48ACDA9 mov       [rsp+18h],rsi
00007FFAC48ACDAE push      rbp
00007FFAC48ACDAF push      rdi
00007FFAC48ACDB0 push      r14
00007FFAC48ACDB2 lea       rbp,[rsp-100h]
00007FFAC48ACDBA sub       rsp,200h
00007FFAC48ACDC1 mov       rax,[rel 7FFA`C475`24E0h]
00007FFAC48ACDC8 xor       rax,rsp
00007FFAC48ACDCB mov       [rbp+0F0h],rax
00007FFAC48ACDD2 mov       r8,[rel 7FFA`C474`F208h]
00007FFAC48ACDD9 lea       rax,[rel 7FFA`C46F`4A58h]
00007FFAC48ACDE0 xor       edi,edi
         */
        static void EncoderExample() {
            var codeReader = new ByteArrayCodeReader(exampleCode);
            var decoder = Decoder.Create(exampleCodeBitness, codeReader);
            decoder.InstructionPointer = exampleCodeRIP;
            ulong endRip = decoder.InstructionPointer + (uint)exampleCode.Length;

            var instructions = new Instruction[exampleCodeNumInstructions];
            int instructionsIndex = 0;
            while (decoder.InstructionPointer < endRip) {
                decoder.Decode(out instructions[instructionsIndex]);
                instructionsIndex++;
            }

            // Relocate the code to some new location. It can fix short/near branches and
            // convert them to short/near/long forms if needed. This also works even if it's a
            // jrcxz/loop/loopcc instruction which only has a short form.
            //
            // It can currently only fix RIP relative operands if the new location is within 2GB
            // of the target data location.
            //
            // There's also a simpler Encoder class which is used by BlockEncoder, but it can only
            // encode one instruction at a time and doesn't fix branches.
            //
            // Note that a block is not the same thing as a basic block. A block can contain any
            // number of instructions, including any number of branch instructions. One block
            // should be enough unless you must relocate different blocks to different locations.
            var codeWriter = new CodeWriterImpl();
            ulong relocatedBaseAddress = exampleCodeRIP + 0x200000;
            var block = new InstructionBlock(codeWriter, instructions, relocatedBaseAddress);
            // This method can also encode more than one block but that's rarely needed, see above comment.
            var errorMessage = BlockEncoder.Encode(decoder.Bitness, block);
            if (errorMessage != null) {
                Console.WriteLine($"ERROR: {errorMessage}");
                return;
            }
            var newCode = codeWriter.ToArray();
            Console.WriteLine("New code bytes:");
            for (int i = 0; i < newCode.Length;) {
                for (int j = 0; j < 16 && i < newCode.Length; i++, j++) {
                    if (j != 0)
                        Console.Write(", ");
                    Console.Write("0x");
                    Console.Write(newCode[i].ToString("X2"));
                }
                Console.WriteLine();
            }

            // Disassemble the new relocated code. It's identical to the original code except that
            // the RIP relative instructions have been updated.
            Console.WriteLine("Disassembled code:");
            var formatter = new NasmFormatter();
            formatter.Options.AddDigitSeparators = true;
            formatter.Options.DigitSeparator = "`";
            formatter.Options.FirstOperandCharIndex = 10;
            var output = new StringBuilderFormatterOutput();
            var newDecoder = Decoder.Create(decoder.Bitness, new ByteArrayCodeReader(newCode));
            newDecoder.InstructionPointer = block.RIP;
            endRip = newDecoder.InstructionPointer + (uint)newCode.Length;
            while (newDecoder.InstructionPointer < endRip) {
                newDecoder.Decode(out var instr);
                formatter.Format(ref instr, output);
                Console.WriteLine($"{instr.IP64:X16} {output.ToStringAndReset()}");
            }
        }
        // Simple and inefficient code writer that stores the data in a List<byte>, with a ToArray() method to get the data
        sealed class CodeWriterImpl : CodeWriter {
            readonly List<byte> allBytes = new List<byte>();
            public override void WriteByte(byte value) => allBytes.Add(value);
            public byte[] ToArray() => allBytes.ToArray();
        }

        /*
         * This method produces the following output:
00007FFAC46ACDA4 mov [rsp+10h],rbx
    Encoding: Legacy
    CpuidFeature: INTEL8086
    FlowControl: Next
    Op0Access: Write
    Op1Access: Read
    RSP:Read
    RBX:Read
    SS:RSP+0x10;UInt64;Write
00007FFAC46ACDA9 mov [rsp+18h],rsi
    Encoding: Legacy
    CpuidFeature: INTEL8086
    FlowControl: Next
    Op0Access: Write
    Op1Access: Read
    RSP:Read
    RSI:Read
    SS:RSP+0x18;UInt64;Write
00007FFAC46ACDAE push rbp
    Encoding: Legacy
    CpuidFeature: INTEL8086
    FlowControl: Next
    SP Increment: -8
    Op0Access: Read
    RBP:Read
    RSP:ReadWrite
    SS:RSP+0xFFFFFFFFFFFFFFF8;UInt64;Write
00007FFAC46ACDAF push rdi
    Encoding: Legacy
    CpuidFeature: INTEL8086
    FlowControl: Next
    SP Increment: -8
    Op0Access: Read
    RDI:Read
    RSP:ReadWrite
    SS:RSP+0xFFFFFFFFFFFFFFF8;UInt64;Write
00007FFAC46ACDB0 push r14
    Encoding: Legacy
    CpuidFeature: INTEL8086
    FlowControl: Next
    SP Increment: -8
    Op0Access: Read
    R14:Read
    RSP:ReadWrite
    SS:RSP+0xFFFFFFFFFFFFFFF8;UInt64;Write
00007FFAC46ACDB2 lea rbp,[rsp-100h]
    Encoding: Legacy
    CpuidFeature: INTEL8086
    FlowControl: Next
    Op0Access: Write
    Op1Access: NoMemAccess
    RBP:Write
    RSP:Read
00007FFAC46ACDBA sub rsp,200h
    Encoding: Legacy
    CpuidFeature: INTEL8086
    FlowControl: Next
    RFLAGS Written: OF, SF, ZF, AF, CF, PF
    RFLAGS Modified: OF, SF, ZF, AF, CF, PF
    Op0Access: ReadWrite
    Op1Access: Read
    RSP:ReadWrite
00007FFAC46ACDC1 mov rax,[7FFAC47524E0h]
    Encoding: Legacy
    CpuidFeature: INTEL8086
    FlowControl: Next
    Op0Access: Write
    Op1Access: Read
    RAX:Write
    DS:0x7FFAC47524E0;UInt64;Read
00007FFAC46ACDC8 xor rax,rsp
    Encoding: Legacy
    CpuidFeature: INTEL8086
    FlowControl: Next
    RFLAGS Written: SF, ZF, PF
    RFLAGS Cleared: OF, CF
    RFLAGS Undefined: AF
    RFLAGS Modified: OF, SF, ZF, AF, CF, PF
    Op0Access: ReadWrite
    Op1Access: Read
    RAX:ReadWrite
    RSP:Read
00007FFAC46ACDCB mov [rbp+0F0h],rax
    Encoding: Legacy
    CpuidFeature: INTEL8086
    FlowControl: Next
    Op0Access: Write
    Op1Access: Read
    RBP:Read
    RAX:Read
    SS:RBP+0xF0;UInt64;Write
00007FFAC46ACDD2 mov r8,[7FFAC474F208h]
    Encoding: Legacy
    CpuidFeature: INTEL8086
    FlowControl: Next
    Op0Access: Write
    Op1Access: Read
    R8:Write
    DS:0x7FFAC474F208;UInt64;Read
00007FFAC46ACDD9 lea rax,[7FFAC46F4A58h]
    Encoding: Legacy
    CpuidFeature: INTEL8086
    FlowControl: Next
    Op0Access: Write
    Op1Access: NoMemAccess
    RAX:Write
00007FFAC46ACDE0 xor edi,edi
    Encoding: Legacy
    CpuidFeature: INTEL8086
    FlowControl: Next
    RFLAGS Written: SF, ZF, PF
    RFLAGS Cleared: OF, CF
    RFLAGS Undefined: AF
    RFLAGS Modified: OF, SF, ZF, AF, CF, PF
    Op0Access: ReadWrite
    Op1Access: Read
    EDI:Read
    RDI:Write
    EDI:Read
         */
        static void InstructionInfoExample() {
            var codeReader = new ByteArrayCodeReader(exampleCode);
            var decoder = Decoder.Create(exampleCodeBitness, codeReader);
            decoder.InstructionPointer = exampleCodeRIP;
            ulong endRip = decoder.InstructionPointer + (uint)exampleCode.Length;

            // For PERF, use a factory to create the instruction info if you need register
            // and memory usage. If it's something else, eg. encoding, flags, etc, there
            // are properties on Instruction that can be used instead that don't allocate.
            // The factory only allocates once and reuses the internal arrays; calling
            // Instruction.GetInfo() allocates every single call.
            var instrInfoFactory = new InstructionInfoFactory();
            while (decoder.InstructionPointer < endRip) {
                decoder.Decode(out var instr);

                // A formatter is recommended since this ToString() method defaults to masm syntax,
                // uses default options, and allocates every single time it's called.
                var disasmStr = instr.ToString();
                Console.WriteLine($"{instr.IP64:X16} {disasmStr}");

                var info = instrInfoFactory.GetInfo(ref instr);
                const string tab = "    ";
                Console.WriteLine($"{tab}Encoding: {info.Encoding}");
                Console.WriteLine($"{tab}CpuidFeature: {info.CpuidFeature}");
                Console.WriteLine($"{tab}FlowControl: {info.FlowControl}");
                if (info.StackInstruction)
                    Console.WriteLine($"{tab}SP Increment: {instr.StackPointerIncrement}");
                if (instr.RflagsRead != RflagsBits.None)
                    Console.WriteLine($"{tab}RFLAGS Read: {instr.RflagsRead}");
                if (instr.RflagsWritten != RflagsBits.None)
                    Console.WriteLine($"{tab}RFLAGS Written: {instr.RflagsWritten}");
                if (instr.RflagsCleared != RflagsBits.None)
                    Console.WriteLine($"{tab}RFLAGS Cleared: {instr.RflagsCleared}");
                if (instr.RflagsSet != RflagsBits.None)
                    Console.WriteLine($"{tab}RFLAGS Set: {instr.RflagsSet}");
                if (instr.RflagsUndefined != RflagsBits.None)
                    Console.WriteLine($"{tab}RFLAGS Undefined: {instr.RflagsUndefined}");
                if (instr.RflagsModified != RflagsBits.None)
                    Console.WriteLine($"{tab}RFLAGS Modified: {instr.RflagsModified}");
                for (int i = 0; i < instr.OpCount; i++)
                    Console.WriteLine($"{tab}Op{i}Access: {info.GetOpAccess(i)}");
                // The returned iterator is a struct, nothing is allocated unless you box it
                foreach (var regInfo in info.GetUsedRegisters())
                    Console.WriteLine($"{tab}{regInfo.ToString()}");
                foreach (var memInfo in info.GetUsedMemory())
                    Console.WriteLine($"{tab}{memInfo.ToString()}");
            }
        }
    }
}
```

# License

LGPL v3 or any later version (LGPL = GNU Lesser General Public License)


# Credits

Logo `processor` by [Creative Stall](https://thenounproject.com/creativestall/) from the Noun Project
