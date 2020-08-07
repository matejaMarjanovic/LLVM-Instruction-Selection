# LLVM-Instruction-Selection

Instruction selection is a phase in the backend that converts IR code to
target-specific code. There are a few implementations of ISel:
- SDAGISel (Selection DAG Instruction Selection)
- FastISel
- GlobalISel


SDAGISel converts IR to SelectionDAG nodes. SelectionDAG nodes are nodes
of the Directed Acyclic Graph. Every DAG represents one basic block. For
every basic block there is a DAG. Nodes represent instructions, variables
and constants. After this pass all IR instructions are converted into
target-specific machine instructions.

After creating a DAG from a basic block, it is necessary to determine the
order of the instructions, because DAG doesn't hold that information. In
order to do that we have to have a three-address representation. The first
step is Pre-register Allocation, it determines the order of the instructions.
Instructions are after that transformed into MachineInstr three-address
representation.

Register Allocation determines the target-specific physical register for
every virtual register. The number of physical registers is finite and
the number of virtual registers can be infinite.

Post-register Allocation uses the knowledge of the special registers and
uses that knowledge so that the efficiency of register allocation is optimal.

Instruction Selection pipeline starts with LLVM IR code from which a DAG that
has IR instructions is made. Then that DAG goes through lowering, where IR
instructions are converted into machine instructions. Then comes the legalization
in which generic machine instructions are transformed into target-specific
instructions. After that we use DAG-to-DAG conversion that transforms SelectionDAG
nodes into nodes that represent the target-specific instructions.



FastISel is an alternative way of implementing ISel. It has better performances,
but is less reliable (SDAGISel would be like -0O compared to FastISel). It skips
the lowering part of the pipeline and converts IR directly to MachineInstr using
TableGen descriptors for simple instructions and a more complex handling for
target-specific instructions.



GlobalISel is better than SDAGISel for many reasons:
- Global, it doesn't work with basic blocks, but the whole function
- Pipeline is more flexible, we can control which target-specific instructions
  will be selected while executing ISel
- Easier to use and maintain
- Contains it's own generic machine representation which is converted to
  target-specific machine representation easily

The first step in GlobalISel is forwarding the IR code to the IRTranslator
which converts it into Generic MIR, which will by the end of this phase become
target-specific MIR. After the IRTranslator comes the Legalizer which replaces
generic machine instructions with target-specific machine instructions. After
that comes the RegBankSelect which assigns virtual registers to Register Banks,
and can also optimize by for example using two 32-bit registers instead of one
64-bit.


GlobalISel
==========

	LLVM IR:
		define double @foo(double %val,
						   double* %addr) {
			%intval = bitcast double %val to i64
			%loaded = load double, double* %addr
			%mask = bitcast double %loaded to i64
			%and = and i64 %intval, %mask
			%res = bitcast i64 %and to double
			ret double %res
		}

	IRTranslator:
		foo:
			val(64) = VMOVDRR R0, R1
			addr(32) = COPY R2
			loaded(64) = gLD (double) addr
			and(64) = gAND (i64) val, loaded
			R0, R1 = VMOVRRD and
			tBX_RET R0<imp-use>,R1<imp-use>

	Legalizer:
		foo:
			val(64) = VMOVDRR R0, R1
			addr(32) = COPY R2
			loaded(64) = gLD (double) addr
			lval(32),hval(32) = extract val
			low(32),high(32) = extract loaded
			land(32) = gAND (i32) lval, low
			hand(32) = gAND (i32) hval, high
			and(64) = build_sequence land, hand
			R0, R1 = VMOVRRD and
			tBX_RET R0<imp-use>,R1<imp-use>

	Legalizer:
		foo:
			val(64) = VMOVDRR R0, R1
			addr(32) = COPY R2
			loaded(64) = gLD (double) addr
			lval(32),hval(32) = VMOVRRD val
			low(32),high(32) = VMOVRRD loaded
			land(32) = gAND (i32) lval, low
			hand(32) = gAND (i32) hval, high
			and(64) = VMOVDRR land, hand
			R0, R1 = VMOVRRD and
			tBX_RET R0<imp-use>,R1<imp-use>

	RegBankSelect:
		foo:
			val(FPR,64) = VMOVDRR R0, R1
			addr(GPR,32) = COPY R2
			loaded(FPR,64) = gLD (double) addr
			lval(GPR,32),hval(GPR,32) = VMOVRRD val
			low(GPR,32),high(GPR,32) = VMOVRRD loaded
			land(GPR,32) = gAND (i32) lval, low
			hand(GPR,32) = gAND (i32) hval, high
			and(FPR,64) = VMOVDRR land, hand
			R0, R1 = VMOVRRD and
			tBX_RET R0<imp-use>,R1<imp-use>

	RegBankSelect:
		foo:
			val1(GPR,32),val2(GPR,32) = VMOVDRR R0, R1
			addr(GPR,32) = COPY R2
			loaded1(GPR,32) = gLD (i32) addr
			loaded2(GPR,32) = gLD (i32) addr, #4
			lval(GPR,32),hval(GPR,32) = COPIES val1, val2
			low(GPR,32),high(GPR,32) = COPIES loaded1, loaded2
			land(GPR,32) = gAND (i32) lval, low
			hand(GPR,32) = gAND (i32) hval, high
			and1(GPR,32),and2(GPR,32) = COPIES land, hand
			R0, R1 = COPIES and1, and2
			tBX_RET R0<imp-use>,R1<imp-use>

	Select:
		foo:
			val1(GPR,32),val2(GPR,32) = VMOVDRR R0, R1
			addr(GPR,32) = COPY R2
			loaded1(GPR,32) = t2LDRi12 (i32) addr
			loaded2(GPR,32) = t2LDRi12 (i32) addr, #4
			lval(GPR,32),hval(GPR,32) = COPIES val1, val2
			low(GPR,32),high(GPR,32) = COPIES loaded1, loaded2
			land(GPR,32) = t2ANDrr (i32) lval, low
			hand(GPR,32) = t2ANDrr (i32) hval, high
			and1(GPR,32),and2(GPR,32) = COPIES land, hand
			R0, R1 = COPIES and1, and2
			tBX_RET R0<imp-use>,R1<imp-use>


SDAGISel
========

	----------------------------------------------------------
	DAG after being built, before the first optimization pass:

		digraph "dag-combine1 input for foo:" {
			graph [bb="0,0,589.5,1102",
				label="dag-combine1 input for foo:",
				lheight=0.21,
				lp="294.75,11.5",
				lwidth=2.15,
				rankdir=BT
			];
			node [label="\N"];
			Node0x6bfc2a8	 [height=0.97222,
				label="{EntryToken|t0|{<d0>ch}}",
				pos="321.5,1067",
				shape=Mrecord,
				width=1.125];
			Node0x6c7b0a0	 [height=0.97222,
				label="{Register %0|t1|{<d0>f64}}",
				pos="321.5,949.5",
				shape=Mrecord,
				width=1.1667];
			Node0x6c7b108	 [height=1.2917,
				label="{{<s0>0|<s1>1}|CopyFromReg|t2|{<d0>f64|<d1>ch}}",
				pos="292.5,820.5",
				shape=Mrecord,
				width=1.3611];
			Node0x6c7b108:s0 -> Node0x6bfc2a8:d0	 [color=blue,
				pos="e,279.5,1044 267.5,867 267.5,942.15 206.77,1034.8 269.35,1043.4",
				style=dashed];
			Node0x6c7b108:s1 -> Node0x6c7b0a0:d0	 [pos="e,321.5,913.5 317.5,867 317.5,883.85 320.14,890.32 321.13,903.28"];
			Node0x6c7b170	 [height=0.97222,
				label="{Register %1|t3|{<d0>i64}}",
				pos="493.5,1067",
				shape=Mrecord,
				width=1.1667];
			Node0x6c7b1d8	 [height=1.2917,
				label="{{<s0>0|<s1>1}|CopyFromReg|t4|{<d0>i64|<d1>ch}}",
				pos="468.5,949.5",
				shape=Mrecord,
				width=1.3611];
			Node0x6c7b1d8:s0 -> Node0x6bfc2a8:d0	 [color=blue,
				pos="e,363.5,1044 418.5,984.5 411.26,984.5 385.89,1023 371.56,1038.1",
				style=dashed];
			Node0x6c7b1d8:s1 -> Node0x6c7b170:d0	 [pos="e,493.5,1032 493.5,996 493.5,1008 493.5,1013.2 493.5,1021.9"];
			Node0x6c7b240	 [height=1.2917,
				label="{{<s0>0}|bitcast|t5|{<d0>i64}}",
				pos="269.5,691.5",
				shape=Mrecord,
				width=0.75];
			Node0x6c7b240:s0 -> Node0x6c7b108:d0	 [pos="e,269.5,774 269.5,738 269.5,750 269.5,755.25 269.5,763.88"];
			Node0x6c7b2a8	 [height=0.97222,
				label="{Constant\<0\>|t6|{<d0>i64}}",
				pos="273.5,58",
				shape=Mrecord,
				width=1.2222];
			Node0x6c7b310	 [height=0.97222,
				label="{undef|t7|{<d0>i64}}",
				pos="562.5,949.5",
				shape=Mrecord,
				width=0.75];
			Node0x6c7b378	 [height=1.2917,
				label="{{<s0>0|<s1>1|<s2>2}|load\<(load 8 from %ir.addr)\>|t8|{<d0>f64|<d1>ch}}",
				pos="451.5,820.5",
				shape=Mrecord,
				width=2.5278];
			Node0x6c7b378:s0 -> Node0x6bfc2a8:d0	 [color=blue,
				pos="e,363.5,1044 390.5,867 390.5,942.85 438.26,1034.9 373.84,1043.4",
				style=dashed];
			Node0x6c7b378:s1 -> Node0x6c7b1d8:d0	 [pos="e,444.5,903 450.5,867 450.5,879.17 447.12,884.21 445.44,892.81"];
			Node0x6c7b378:s2 -> Node0x6c7b310:d0	 [pos="e,534.5,926.5 543.5,855.5 555.74,855.5 534.39,897.54 531.15,916.92"];
			Node0x6c7b3e0	 [height=1.2917,
				label="{{<s0>0}|bitcast|t9|{<d0>i64}}",
				pos="373.5,691.5",
				shape=Mrecord,
				width=0.75];
			Node0x6c7b3e0:s0 -> Node0x6c7b378:d0	 [pos="e,407.5,774 401.5,726.5 419.45,726.5 411.38,745.17 408.42,763.78"];
			Node0x6c7b448	 [height=1.2917,
				label="{{<s0>0|<s1>1}|and|t10|{<d0>i64}}",
				pos="283.5,562.5",
				shape=Mrecord,
				width=0.75];
			Node0x6c7b448:s0 -> Node0x6c7b240:d0	 [pos="e,269.5,645 269.5,609 269.5,621 269.5,626.25 269.5,634.88"];
			Node0x6c7b448:s1 -> Node0x6c7b3e0:d0	 [pos="e,345.5,656.5 297.5,609 297.5,635.5 311.52,652.23 335.45,655.79"];
			Node0x6c7b4b0	 [height=1.2917,
				label="{{<s0>0}|bitcast|t11|{<d0>f64}}",
				pos="276.5,433.5",
				shape=Mrecord,
				width=0.75];
			Node0x6c7b4b0:s0 -> Node0x6c7b448:d0	 [pos="e,283.5,516 276.5,480 276.5,492.22 280.44,497.19 282.41,505.79"];
			Node0x6c7b518	 [height=0.97222,
				label="{TargetConstant\<0\>|t12|{<d0>i32}}",
				pos="61.5,304.5",
				shape=Mrecord,
				width=1.7083];
			Node0x6c7b580	 [height=0.97222,
				label="{Register $xmm0|t13|{<d0>f64}}",
				pos="131.5,433.5",
				shape=Mrecord,
				width=1.5];
			Node0x6c7b5e8	 [height=1.2917,
				label="{{<s0>0|<s1>1|<s2>2}|CopyToReg|t14|{<d0>ch|<d1>glue}}",
				pos="221.5,304.5",
				shape=Mrecord,
				width=1.1528];
			Node0x6c7b5e8:s0 -> Node0x6bfc2a8:d0	 [color=blue,
				pos="e,279.5,1044 193.5,351 193.5,444.98 213.5,467.52 213.5,561.5 213.5,561.5 213.5,561.5 213.5,821.5 213.5,921.12 178.85,1035.9 269.41,\
		1043.6",
				style=dashed];
			Node0x6c7b5e8:s2 -> Node0x6c7b4b0:d0	 [pos="e,248.5,398.5 249.5,351 249.5,366.67 237.32,385.25 239.67,393.78"];
			Node0x6c7b5e8:s1 -> Node0x6c7b580:d0	 [pos="e,186.5,410.5 220.5,351 220.5,369.52 208.65,370.89 199.5,387 196.09,393 196.27,399.92 194.72,404.68"];
			Node0x6c7b650	 [height=1.2917,
				label="{{<s0>0|<s1>1|<s2>2|<s3>3}|X86ISD::RET_FLAG|t15|{<d0>ch}}",
				pos="161.5,175.5",
				shape=Mrecord,
				width=1.9028];
			Node0x6c7b650:s1 -> Node0x6c7b518:d0	 [pos="e,124.5,281.5 143.5,222 143.5,227.2 136.72,258.07 130.4,273.18"];
			Node0x6c7b650:s2 -> Node0x6c7b580:d0	 [pos="e,131.5,397.5 178.5,222 178.5,240.06 140.89,350.09 132.96,387.55"];
			Node0x6c7b650:s0 -> Node0x6c7b5e8:d0	 [color=blue,
				pos="e,197.5,258 109.5,222 109.5,260.13 181.15,224.35 195.15,248.05",
				style=dashed];
			Node0x6c7b650:s3 -> Node0x6c7b5e8:d1	 [color=red,
				pos="e,239.5,258 212.5,222 212.5,237.94 229.65,238.63 236.61,248.38",
				style=bold];
			Node0x0	 [height=0.5,
				label=GraphRoot,
				plaintext=circle,
				pos="161.5,58",
				width=1.3902];
			Node0x0 -> Node0x6c7b650:d0	 [color=blue,
				pos="e,161.5,129 161.5,76.26 161.5,87.772 161.5,103.52 161.5,118.67",
				style=dashed];
		}

	------------------------
	DAG before Legalization:


		digraph "legalize input for foo:" {
			graph [bb="0,0,589.5,810",
				label="legalize input for foo:",
				lheight=0.21,
				lp="294.75,11.5",
				lwidth=1.65,
				rankdir=BT
			];
			node [label="\N"];
			Node0x70f12a8	 [height=0.97222,
				label="{EntryToken|t0|{<d0>ch}}",
				pos="321.5,775",
				shape=Mrecord,
				width=1.125];
			Node0x71700a0	 [height=0.97222,
				label="{Register %0|t1|{<d0>f64}}",
				pos="321.5,657.5",
				shape=Mrecord,
				width=1.1667];
			Node0x7170108	 [height=1.2917,
				label="{{<s0>0|<s1>1}|CopyFromReg|t2|{<d0>f64|<d1>ch}}",
				pos="291.5,528.5",
				shape=Mrecord,
				width=1.3611];
			Node0x7170108:s0 -> Node0x70f12a8:d0	 [color=blue,
				pos="e,279.5,752 266.5,575 266.5,650.18 206.65,742.84 269.34,751.37",
				style=dashed];
			Node0x7170108:s1 -> Node0x71700a0:d0	 [pos="e,321.5,621.5 316.5,575 316.5,591.89 319.8,598.31 321.04,611.27"];
			Node0x7170170	 [height=0.97222,
				label="{Register %1|t3|{<d0>i64}}",
				pos="493.5,775",
				shape=Mrecord,
				width=1.1667];
			Node0x71701d8	 [height=1.2917,
				label="{{<s0>0|<s1>1}|CopyFromReg|t4|{<d0>i64|<d1>ch}}",
				pos="468.5,657.5",
				shape=Mrecord,
				width=1.3611];
			Node0x71701d8:s0 -> Node0x70f12a8:d0	 [color=blue,
				pos="e,363.5,752 418.5,692.5 411.26,692.5 385.89,731.03 371.56,746.08",
				style=dashed];
			Node0x71701d8:s1 -> Node0x7170170:d0	 [pos="e,493.5,740 493.5,704 493.5,716 493.5,721.25 493.5,729.88"];
			Node0x7170310	 [height=0.97222,
				label="{undef|t7|{<d0>i64}}",
				pos="562.5,657.5",
				shape=Mrecord,
				width=0.75];
			Node0x7170378	 [height=1.2917,
				label="{{<s0>0|<s1>1|<s2>2}|load\<(load 8 from %ir.addr)\>|t8|{<d0>f64|<d1>ch}}",
				pos="451.5,528.5",
				shape=Mrecord,
				width=2.5278];
			Node0x7170378:s0 -> Node0x70f12a8:d0	 [color=blue,
				pos="e,363.5,752 390.5,575 390.5,650.85 438.26,742.91 373.84,751.37",
				style=dashed];
			Node0x7170378:s1 -> Node0x71701d8:d0	 [pos="e,444.5,611 450.5,575 450.5,587.17 447.12,592.21 445.44,600.81"];
			Node0x7170378:s2 -> Node0x7170310:d0	 [pos="e,534.5,634.5 543.5,563.5 555.74,563.5 534.39,605.54 531.15,624.92"];
			Node0x7170518	 [height=0.97222,
				label="{TargetConstant\<0\>|t12|{<d0>i32}}",
				pos="61.5,270.5",
				shape=Mrecord,
				width=1.7083];
			Node0x7170580	 [height=0.97222,
				label="{Register $xmm0|t13|{<d0>f64}}",
				pos="131.5,399.5",
				shape=Mrecord,
				width=1.5];
			Node0x71705e8	 [height=1.2917,
				label="{{<s0>0|<s1>1|<s2>2}|CopyToReg|t14|{<d0>ch|<d1>glue}}",
				pos="221.5,270.5",
				shape=Mrecord,
				width=1.1528];
			Node0x71705e8:s0 -> Node0x70f12a8:d0	 [color=blue,
				pos="e,279.5,752 193.5,317 193.5,360.29 221.19,664.32 238.5,704 248.18,726.2 250.79,745.84 269.38,750.8",
				style=dashed];
			Node0x71705e8:s1 -> Node0x7170580:d0	 [pos="e,186.5,376.5 220.5,317 220.5,335.52 208.65,336.89 199.5,353 196.09,359 196.27,365.92 194.72,370.68"];
			Node0x71702a8	 [height=1.2917,
				label="{{<s0>0|<s1>1}|X86ISD::FAND|t16|{<d0>f64}}",
				pos="295.5,399.5",
				shape=Mrecord,
				width=1.4722];
			Node0x71705e8:s2 -> Node0x71702a8:d0	 [pos="e,241.5,364.5 249.5,317 249.5,331.72 235.6,348.65 233.87,357.8"];
			Node0x71702a8:s0 -> Node0x7170108:d0	 [pos="e,268.5,482 268.5,446 268.5,458 268.5,463.25 268.5,471.88"];
			Node0x71702a8:s1 -> Node0x7170378:d0	 [pos="e,359.5,493.5 349.5,434.5 371.11,434.5 346.65,473.45 350.98,488.06"];
			Node0x7170650	 [height=1.2917,
				label="{{<s0>0|<s1>1|<s2>2|<s3>3}|X86ISD::RET_FLAG|t15|{<d0>ch}}",
				pos="161.5,141.5",
				shape=Mrecord,
				width=1.9028];
			Node0x7170650:s1 -> Node0x7170518:d0	 [pos="e,124.5,247.5 143.5,188 143.5,193.2 136.72,224.07 130.4,239.18"];
			Node0x7170650:s2 -> Node0x7170580:d0	 [pos="e,131.5,363.5 178.5,188 178.5,206.06 140.89,316.09 132.96,353.55"];
			Node0x7170650:s0 -> Node0x71705e8:d0	 [color=blue,
				pos="e,197.5,224 109.5,188 109.5,226.13 181.15,190.35 195.15,214.05",
				style=dashed];
			Node0x7170650:s3 -> Node0x71705e8:d1	 [color=red,
				pos="e,239.5,224 212.5,188 212.5,203.94 229.65,204.63 236.61,214.38",
				style=bold];
			Node0x0	 [height=0.5,
				label=GraphRoot,
				plaintext=circle,
				pos="161.5,41",
				width=1.3902];
			Node0x0 -> Node0x7170650:d0	 [color=blue,
				pos="e,161.5,95 161.5,59.186 161.5,66.722 161.5,75.899 161.5,84.942",
				style=dashed];
		}


	----------------------------------------
	DAG before the second optimization pass:

		digraph "dag-combine2 input for foo:" {
			graph [bb="0,0,589.5,810",
				label="dag-combine2 input for foo:",
				lheight=0.21,
				lp="294.75,11.5",
				lwidth=2.15,
				rankdir=BT
			];
			node [label="\N"];
			Node0x59b12a8	 [height=0.97222,
				label="{EntryToken|t0|{<d0>ch}}",
				pos="321.5,775",
				shape=Mrecord,
				width=1.125];
			Node0x5a300a0	 [height=0.97222,
				label="{Register %0|t1|{<d0>f64}}",
				pos="321.5,657.5",
				shape=Mrecord,
				width=1.1667];
			Node0x5a30170	 [height=0.97222,
				label="{Register %1|t3|{<d0>i64}}",
				pos="493.5,775",
				shape=Mrecord,
				width=1.1667];
			Node0x5a30310	 [height=0.97222,
				label="{undef|t7|{<d0>i64}}",
				pos="562.5,657.5",
				shape=Mrecord,
				width=0.75];
			Node0x5a30518	 [height=0.97222,
				label="{TargetConstant\<0\>|t12|{<d0>i32}}",
				pos="61.5,270.5",
				shape=Mrecord,
				width=1.7083];
			Node0x5a30580	 [height=0.97222,
				label="{Register $xmm0|t13|{<d0>f64}}",
				pos="131.5,399.5",
				shape=Mrecord,
				width=1.5];
			Node0x5a30108	 [height=1.2917,
				label="{{<s0>0|<s1>1}|CopyFromReg|t2|{<d0>f64|<d1>ch}}",
				pos="291.5,528.5",
				shape=Mrecord,
				width=1.3611];
			Node0x5a30108:s0 -> Node0x59b12a8:d0	 [color=blue,
				pos="e,279.5,752 266.5,575 266.5,650.18 206.65,742.84 269.34,751.37",
				style=dashed];
			Node0x5a30108:s1 -> Node0x5a300a0:d0	 [pos="e,321.5,621.5 316.5,575 316.5,591.89 319.8,598.31 321.04,611.27"];
			Node0x5a301d8	 [height=1.2917,
				label="{{<s0>0|<s1>1}|CopyFromReg|t4|{<d0>i64|<d1>ch}}",
				pos="468.5,657.5",
				shape=Mrecord,
				width=1.3611];
			Node0x5a301d8:s0 -> Node0x59b12a8:d0	 [color=blue,
				pos="e,363.5,752 418.5,692.5 411.26,692.5 385.89,731.03 371.56,746.08",
				style=dashed];
			Node0x5a301d8:s1 -> Node0x5a30170:d0	 [pos="e,493.5,740 493.5,704 493.5,716 493.5,721.25 493.5,729.88"];
			Node0x5a30378	 [height=1.2917,
				label="{{<s0>0|<s1>1|<s2>2}|load\<(load 8 from %ir.addr)\>|t8|{<d0>f64|<d1>ch}}",
				pos="451.5,528.5",
				shape=Mrecord,
				width=2.5278];
			Node0x5a30378:s0 -> Node0x59b12a8:d0	 [color=blue,
				pos="e,363.5,752 390.5,575 390.5,650.85 438.26,742.91 373.84,751.37",
				style=dashed];
			Node0x5a30378:s2 -> Node0x5a30310:d0	 [pos="e,534.5,634.5 543.5,563.5 555.74,563.5 534.39,605.54 531.15,624.92"];
			Node0x5a30378:s1 -> Node0x5a301d8:d0	 [pos="e,444.5,611 450.5,575 450.5,587.17 447.12,592.21 445.44,600.81"];
			Node0x5a302a8	 [height=1.2917,
				label="{{<s0>0|<s1>1}|X86ISD::FAND|t16|{<d0>f64}}",
				pos="295.5,399.5",
				shape=Mrecord,
				width=1.4722];
			Node0x5a302a8:s0 -> Node0x5a30108:d0	 [pos="e,268.5,482 268.5,446 268.5,458 268.5,463.25 268.5,471.88"];
			Node0x5a302a8:s1 -> Node0x5a30378:d0	 [pos="e,359.5,493.5 349.5,434.5 371.11,434.5 346.65,473.45 350.98,488.06"];
			Node0x5a305e8	 [height=1.2917,
				label="{{<s0>0|<s1>1|<s2>2}|CopyToReg|t14|{<d0>ch|<d1>glue}}",
				pos="221.5,270.5",
				shape=Mrecord,
				width=1.1528];
			Node0x5a305e8:s0 -> Node0x59b12a8:d0	 [color=blue,
				pos="e,279.5,752 193.5,317 193.5,360.29 221.19,664.32 238.5,704 248.18,726.2 250.79,745.84 269.38,750.8",
				style=dashed];
			Node0x5a305e8:s1 -> Node0x5a30580:d0	 [pos="e,186.5,376.5 220.5,317 220.5,335.52 208.65,336.89 199.5,353 196.09,359 196.27,365.92 194.72,370.68"];
			Node0x5a305e8:s2 -> Node0x5a302a8:d0	 [pos="e,241.5,364.5 249.5,317 249.5,331.72 235.6,348.65 233.87,357.8"];
			Node0x5a30650	 [height=1.2917,
				label="{{<s0>0|<s1>1|<s2>2|<s3>3}|X86ISD::RET_FLAG|t15|{<d0>ch}}",
				pos="161.5,141.5",
				shape=Mrecord,
				width=1.9028];
			Node0x5a30650:s1 -> Node0x5a30518:d0	 [pos="e,124.5,247.5 143.5,188 143.5,193.2 136.72,224.07 130.4,239.18"];
			Node0x5a30650:s2 -> Node0x5a30580:d0	 [pos="e,131.5,363.5 178.5,188 178.5,206.06 140.89,316.09 132.96,353.55"];
			Node0x5a30650:s0 -> Node0x5a305e8:d0	 [color=blue,
				pos="e,197.5,224 109.5,188 109.5,226.13 181.15,190.35 195.15,214.05",
				style=dashed];
			Node0x5a30650:s3 -> Node0x5a305e8:d1	 [color=red,
				pos="e,239.5,224 212.5,188 212.5,203.94 229.65,204.63 236.61,214.38",
				style=bold];
			Node0x0	 [height=0.5,
				label=GraphRoot,
				plaintext=circle,
				pos="161.5,41",
				width=1.3902];
			Node0x0 -> Node0x5a30650:d0	 [color=blue,
				pos="e,161.5,95 161.5,59.186 161.5,66.722 161.5,75.899 161.5,84.942",
				style=dashed];
		}


	----------------------------
	DAG before the Select phase:

		digraph "isel input for foo:" {
			graph [bb="0,0,589.5,810",
				label="isel input for foo:",
				lheight=0.21,
				lp="294.75,11.5",
				lwidth=1.33,
				rankdir=BT
			];
			node [label="\N"];
			Node0x5fb62a8	 [height=0.97222,
				label="{EntryToken|t0|{<d0>ch}}",
				pos="321.5,775",
				shape=Mrecord,
				width=1.125];
			Node0x60350a0	 [height=0.97222,
				label="{Register %0|t1|{<d0>f64}}",
				pos="321.5,657.5",
				shape=Mrecord,
				width=1.1667];
			Node0x6035170	 [height=0.97222,
				label="{Register %1|t3|{<d0>i64}}",
				pos="493.5,775",
				shape=Mrecord,
				width=1.1667];
			Node0x6035310	 [height=0.97222,
				label="{undef|t7|{<d0>i64}}",
				pos="562.5,657.5",
				shape=Mrecord,
				width=0.75];
			Node0x6035518	 [height=0.97222,
				label="{TargetConstant\<0\>|t12|{<d0>i32}}",
				pos="61.5,270.5",
				shape=Mrecord,
				width=1.7083];
			Node0x6035580	 [height=0.97222,
				label="{Register $xmm0|t13|{<d0>f64}}",
				pos="131.5,399.5",
				shape=Mrecord,
				width=1.5];
			Node0x6035108	 [height=1.2917,
				label="{{<s0>0|<s1>1}|CopyFromReg|t2|{<d0>f64|<d1>ch}}",
				pos="291.5,528.5",
				shape=Mrecord,
				width=1.3611];
			Node0x6035108:s0 -> Node0x5fb62a8:d0	 [color=blue,
				pos="e,279.5,752 266.5,575 266.5,650.18 206.65,742.84 269.34,751.37",
				style=dashed];
			Node0x6035108:s1 -> Node0x60350a0:d0	 [pos="e,321.5,621.5 316.5,575 316.5,591.89 319.8,598.31 321.04,611.27"];
			Node0x60351d8	 [height=1.2917,
				label="{{<s0>0|<s1>1}|CopyFromReg|t4|{<d0>i64|<d1>ch}}",
				pos="468.5,657.5",
				shape=Mrecord,
				width=1.3611];
			Node0x60351d8:s0 -> Node0x5fb62a8:d0	 [color=blue,
				pos="e,363.5,752 418.5,692.5 411.26,692.5 385.89,731.03 371.56,746.08",
				style=dashed];
			Node0x60351d8:s1 -> Node0x6035170:d0	 [pos="e,493.5,740 493.5,704 493.5,716 493.5,721.25 493.5,729.88"];
			Node0x6035378	 [height=1.2917,
				label="{{<s0>0|<s1>1|<s2>2}|load\<(load 8 from %ir.addr)\>|t8|{<d0>f64|<d1>ch}}",
				pos="451.5,528.5",
				shape=Mrecord,
				width=2.5278];
			Node0x6035378:s0 -> Node0x5fb62a8:d0	 [color=blue,
				pos="e,363.5,752 390.5,575 390.5,650.85 438.26,742.91 373.84,751.37",
				style=dashed];
			Node0x6035378:s2 -> Node0x6035310:d0	 [pos="e,534.5,634.5 543.5,563.5 555.74,563.5 534.39,605.54 531.15,624.92"];
			Node0x6035378:s1 -> Node0x60351d8:d0	 [pos="e,444.5,611 450.5,575 450.5,587.17 447.12,592.21 445.44,600.81"];
			Node0x60352a8	 [height=1.2917,
				label="{{<s0>0|<s1>1}|X86ISD::FAND|t16|{<d0>f64}}",
				pos="295.5,399.5",
				shape=Mrecord,
				width=1.4722];
			Node0x60352a8:s0 -> Node0x6035108:d0	 [pos="e,268.5,482 268.5,446 268.5,458 268.5,463.25 268.5,471.88"];
			Node0x60352a8:s1 -> Node0x6035378:d0	 [pos="e,359.5,493.5 349.5,434.5 371.11,434.5 346.65,473.45 350.98,488.06"];
			Node0x60355e8	 [height=1.2917,
				label="{{<s0>0|<s1>1|<s2>2}|CopyToReg|t14|{<d0>ch|<d1>glue}}",
				pos="221.5,270.5",
				shape=Mrecord,
				width=1.1528];
			Node0x60355e8:s0 -> Node0x5fb62a8:d0	 [color=blue,
				pos="e,279.5,752 193.5,317 193.5,360.29 221.19,664.32 238.5,704 248.18,726.2 250.79,745.84 269.38,750.8",
				style=dashed];
			Node0x60355e8:s1 -> Node0x6035580:d0	 [pos="e,186.5,376.5 220.5,317 220.5,335.52 208.65,336.89 199.5,353 196.09,359 196.27,365.92 194.72,370.68"];
			Node0x60355e8:s2 -> Node0x60352a8:d0	 [pos="e,241.5,364.5 249.5,317 249.5,331.72 235.6,348.65 233.87,357.8"];
			Node0x6035650	 [height=1.2917,
				label="{{<s0>0|<s1>1|<s2>2|<s3>3}|X86ISD::RET_FLAG|t15|{<d0>ch}}",
				pos="161.5,141.5",
				shape=Mrecord,
				width=1.9028];
			Node0x6035650:s1 -> Node0x6035518:d0	 [pos="e,124.5,247.5 143.5,188 143.5,193.2 136.72,224.07 130.4,239.18"];
			Node0x6035650:s2 -> Node0x6035580:d0	 [pos="e,131.5,363.5 178.5,188 178.5,206.06 140.89,316.09 132.96,353.55"];
			Node0x6035650:s0 -> Node0x60355e8:d0	 [color=blue,
				pos="e,197.5,224 109.5,188 109.5,226.13 181.15,190.35 195.15,214.05",
				style=dashed];
			Node0x6035650:s3 -> Node0x60355e8:d1	 [color=red,
				pos="e,239.5,224 212.5,188 212.5,203.94 229.65,204.63 236.61,214.38",
				style=bold];
			Node0x0	 [height=0.5,
				label=GraphRoot,
				plaintext=circle,
				pos="161.5,41",
				width=1.3902];
			Node0x0 -> Node0x6035650:d0	 [color=blue,
				pos="e,161.5,95 161.5,59.186 161.5,66.722 161.5,75.899 161.5,84.942",
				style=dashed];
		}


	----------------------
	DAG before Scheduling:

		digraph "scheduler input for foo:" {
			graph [bb="0,0,962.22,939",
				label="scheduler input for foo:",
				lheight=0.21,
				lp="481.11,11.5",
				lwidth=1.79,
				rankdir=BT
			];
			node [label="\N"];
			Node0x792e2a8	 [height=0.97222,
				label="{EntryToken|t0|{<d0>ch}}",
				pos="229,904",
				shape=Mrecord,
				width=1.125];
			Node0x79ad0a0	 [height=0.97222,
				label="{Register %0|t1|{<d0>f64}}",
				pos="74,904",
				shape=Mrecord,
				width=1.1667];
			Node0x79ad170	 [height=0.97222,
				label="{Register %1|t3|{<d0>i64}}",
				pos="384,904",
				shape=Mrecord,
				width=1.1667];
			Node0x79ad518	 [height=0.97222,
				label="{TargetConstant\<0\>|t12|{<d0>i32}}",
				pos="750,786.5",
				shape=Mrecord,
				width=1.7083];
			Node0x79ad580	 [height=0.97222,
				label="{Register $xmm0|t13|{<d0>f64}}",
				pos="209,399.5",
				shape=Mrecord,
				width=1.5];
			Node0x79ad108	 [height=1.2917,
				label="{{<s0>0|<s1>1}|CopyFromReg|t2|{<d0>f64|<d1>ch}}",
				pos="49,786.5",
				shape=Mrecord,
				width=1.3611];
			Node0x79ad108:s0 -> Node0x792e2a8:d0	 [color=blue,
				pos="e,187,881 24,833 24,841.22 140.52,871.54 176.8,879.22",
				style=dashed];
			Node0x79ad108:s1 -> Node0x79ad0a0:d0	 [pos="e,74,869 74,833 74,845 74,850.25 74,858.88"];
			Node0x79ad1d8	 [height=1.2917,
				label="{{<s0>0|<s1>1}|CopyFromReg|t4|{<d0>i64|<d1>ch}}",
				pos="359,786.5",
				shape=Mrecord,
				width=1.3611];
			Node0x79ad1d8:s0 -> Node0x792e2a8:d0	 [color=blue,
				pos="e,271,881 309,821.5 306,821.5 287.62,856.38 277.41,872.73",
				style=dashed];
			Node0x79ad1d8:s1 -> Node0x79ad170:d0	 [pos="e,384,869 384,833 384,845 384,850.25 384,858.88"];
			Node0x79ad240	 [height=1.2917,
				label="{{<s0>0|<s1>1}|COPY_TO_REGCLASS|t17|{<d0>v2f64}}",
				pos="231,657.5",
				shape=Mrecord,
				width=2.125];
			Node0x79ad240:s0 -> Node0x79ad108:d0	 [pos="e,26,740 153,692.5 126.28,692.5 47.09,706.16 29.491,730.26"];
			Node0x79ad448	 [height=0.97222,
				label="{TargetConstant\<112\>|t29|{<d0>i32}}",
				pos="223,786.5",
				shape=Mrecord,
				width=1.9028];
			Node0x79ad240:s1 -> Node0x79ad448:d0	 [pos="e,223,750.5 269,704 269,728.98 235.03,723.9 225.47,740.42"];
			Node0x79ad3e0	 [height=1.2917,
				label="{{<s0>0|<s1>1|<s2>2|<s3>3|<s4>4|<s5>5}|MOVSDrm\<Mem:(load 8 from %ir.addr)\>|t18|{<d0>v2f64|<d1>ch}}",
				pos="641,657.5",
				shape=Mrecord,
				width=3.5694];
			Node0x79ad3e0:s5 -> Node0x792e2a8:d0	 [color=blue,
				pos="e,271,881 771,692.5 810.44,692.5 918.13,708.6 942,740 967.01,772.91 970.35,802.92 942,833 895.5,882.33 400.25,860.54 333,869 308.85,\
		872.04 301.07,878.83 281.12,880.58",
				style=dashed];
			Node0x79ad3e0:s3 -> Node0x79ad518:d0	 [pos="e,687,763.5 662,704 662,721.89 673.42,723.36 680,740 682,745.07 681.03,750.94 680.92,755.55"];
			Node0x79ad3e0:s0 -> Node0x79ad1d8:d0	 [pos="e,335,740 511,692.5 473.97,692.5 357.59,698.34 337.86,730.28"];
			Node0x79ad4b0	 [height=0.97222,
				label="{TargetConstant\<1\>|t26|{<d0>i8}}",
				pos="488,786.5",
				shape=Mrecord,
				width=1.7083];
			Node0x79ad3e0:s1 -> Node0x79ad4b0:d0	 [pos="e,551,763.5 576,704 576,721.69 565.64,723.6 559,740 556.93,745.11 557.69,750.99 557.58,755.59"];
			Node0x79ad720	 [height=0.97222,
				label="{Register $noreg|t27|{<d0>i64}}",
				pos="619,786.5",
				shape=Mrecord,
				width=1.4306];
			Node0x79ad3e0:s2 -> Node0x79ad720:d0	 [pos="e,619,750.5 619,704 619,720.79 619,727.35 619,740.31"];
			Node0x79ad788	 [height=0.97222,
				label="{Register $noreg|t28|{<d0>i16}}",
				pos="881,786.5",
				shape=Mrecord,
				width=1.4306];
			Node0x79ad3e0:s4 -> Node0x79ad788:d0	 [pos="e,828,763.5 705,704 705,757.98 783.05,701.61 821,740 824.83,743.87 823.86,749.75 823.29,754.66"];
			Node0x79ad6b8	 [height=1.2917,
				label="{{<s0>0|<s1>1}|PANDrr|t21|{<d0>v2i64}}",
				pos="338,528.5",
				shape=Mrecord,
				width=0.86111];
			Node0x79ad6b8:s0 -> Node0x79ad240:d0	 [pos="e,309,622.5 306,563.5 285.49,563.5 314.88,599.51 316.2,615.27"];
			Node0x79ad6b8:s1 -> Node0x79ad3e0:d0	 [pos="e,511,622.5 370,563.5 376.23,563.5 373.05,571.22 378,575 425.03,610.98 445.83,621.35 500.73,622.41"];
			Node0x79ad7f0	 [height=1.2917,
				label="{{<s0>0|<s1>1}|COPY_TO_REGCLASS|t24|{<d0>f64}}",
				pos="396,399.5",
				shape=Mrecord,
				width=2.125];
			Node0x79ad7f0:s0 -> Node0x79ad6b8:d0	 [pos="e,370,493.5 357,446 357,460.36 372.02,476.33 376.17,485.61"];
			Node0x79ad2a8	 [height=0.97222,
				label="{TargetConstant\<67\>|t25|{<d0>i32}}",
				pos="452,528.5",
				shape=Mrecord,
				width=1.8056];
			Node0x79ad7f0:s1 -> Node0x79ad2a8:d0	 [pos="e,386,505.5 434,446 434,475.7 399.42,466.52 387,493.5 386.72,494.11 386.41,494.75 386.09,495.41"];
			Node0x79ad5e8	 [height=1.2917,
				label="{{<s0>0|<s1>1|<s2>2}|CopyToReg|t14|{<d0>ch|<d1>glue}}",
				pos="279,270.5",
				shape=Mrecord,
				width=1.1528];
			Node0x79ad5e8:s0 -> Node0x792e2a8:d0	 [color=blue,
				pos="e,187,881 251,317 251,335.52 267.17,335.12 272,353 282.79,392.9 285.32,406.87 272,446 242.19,533.61 174.81,523.39 145,611 113.22,\
		704.41 110.3,740.64 145,833 153.65,856.01 157.56,875.21 176.94,879.9",
				style=dashed];
			Node0x79ad5e8:s1 -> Node0x79ad580:d0	 [pos="e,264,376.5 278,317 278,327.29 277.76,356.16 271.73,369.53"];
			Node0x79ad5e8:s2 -> Node0x79ad7f0:d0	 [pos="e,318,364.5 322,305.5 342.53,305.5 312.5,341.51 310.91,357.27"];
			Node0x79ad650	 [height=1.2917,
				label="{{<s0>0|<s1>1|<s2>2|<s3>3}|RET|t15|{<d0>ch}}",
				pos="232,141.5",
				shape=Mrecord,
				width=1.2778];
			Node0x79ad650:s0 -> Node0x79ad518:d0	 [pos="e,687,763.5 197,188 197,304.89 96.899,339.93 146,446 183.22,526.41 226.49,522.68 298,575 385.19,638.78 403.71,663.87 504,704 578.13,\
		733.66 621.12,686.07 680,740 684.02,743.68 683.04,749.56 682.43,754.51"];
			Node0x79ad650:s1 -> Node0x79ad580:d0	 [pos="e,209,363.5 220,188 220,262.64 209.97,283.5 209.06,353.31"];
			Node0x79ad650:s2 -> Node0x79ad5e8:d0	 [color=blue,
				pos="e,255,224 244,188 244,200.68 250.32,205.21 253.38,213.94",
				style=dashed];
			Node0x79ad650:s3 -> Node0x79ad5e8:d1	 [color=red,
				pos="e,297,224 267,188 267,204.76 286.43,204.37 294.01,214.43",
				style=bold];
			Node0x0	 [height=0.5,
				label=GraphRoot,
				plaintext=circle,
				pos="232,41",
				width=1.3902];
			Node0x0 -> Node0x79ad650:d0	 [color=blue,
				pos="e,232,95 232,59.186 232,66.722 232,75.899 232,84.942",
				style=dashed];
		}


References:

[0] LLVM cookbook : over 80 engaging recipes that will help you build a compiler
frontend, optimizer, and code generator using LLVM by Mayur Pandey, Suyog Sarda
https://github.com/iBreaker/book/blob/master/LLVM%20Cookbook.pdf

[1] Getting Started with LLVM Core Libraries: Get to Grips with LLVM Essentials
and Use the Core Libraries to Build Advanced Tools by Bruno Cardoso Lopes, Rafael Auler
http://faculty.sist.shanghaitech.edu.cn/faculty/songfu/course/spring2018/CS131/llvm.pdf

[2] YouTube presentation 2008 LLVM Developers’ Meeting: D. Gohman “CodeGen Overview
and Focus on SelectionDAGs” https://www.youtube.com/watch?v=Y5kbSDi7qb8&t=469s

[3] YouTube presentation 2015 LLVM Developers’ Meeting: Quentin Colombet “A Proposal
for Global Instruction Selection" https://www.youtube.com/watch?v=F6GGbYtae3g&t=1706s

[4] YouTube presentation 2019 LLVM Developers’ Meeting: V. Keles & D. Sanders
“Generating Optimized Code with GlobalISel” https://www.youtube.com/watch?v=8427bl_7k1g

