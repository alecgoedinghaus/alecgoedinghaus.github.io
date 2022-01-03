---
title:  "RISC-V CPU Simulator"
mathjax: true
layout: post
categories: [CS, C++]
---

## Overview

This project began as two separate projects for my computer architecture course (CS M151B / ECE M116C).

The goal of the first project was to simulate a pipelined RISC-V CPU, capable of handing nine different instructions: `SUB`, `ADD`, `OR`, `AND`, `ADDI`, `ORI`, `ANDI`, `LW`, and `SW`.

The goal of the second project was to simulate a CPU cache with three different sorting policies: direct-mapped, fully associative, and set associative.
Each policy has its pro's and con's, as the access time for a DM cache is much lower than SA and FA, but the miss rate for a DM cache is much higher than SA and FA.

After the course I took the liberty of merging the two projects together, thus allowing the cache to give the CPU better performance.

More information about RISC-V's ISA and cache replacement policies can be found [here](https://riscv.org/wp-content/uploads/2017/05/riscv-spec-v2.2.pdf) and [here](https://en.wikipedia.org/wiki/Cache_replacement_policies), respectively.

## Running

The program can be run in the command line.
It takes two arguments: the path to the text file containing assembly instructions, and a value 0, 1, or 2 corresponding to the DM, FA, and SA cache type, respectively.

After execution, the program will output the values of registers `a0` and `a1`.
It will also print other statistics, as shown below:
```bash
% ./cpusim ./text_files/instMem-t.txt 0           
(a0, a1):        (9, 11)
Cache Type:      DM
Clock:           33
LW Miss Rate:    1.0000
SW Miss Rate:    1.0000
```

## Text Files

The text file must be the translated binary representation of the RISC-V assembly, with each line representing a byte.
For example,
```
#test:
	0:		00200293		addi x5 x0 2
	4:		00300313		addi x6 x0 3
	8:		00002223		sw x0 4 x0
	c:		00002423		sw x0 8 x0
	10:		00502623		sw x5 12 x0
	14:		00602823		sw x6 16 x0
	18:		00402283		lw x5 4 x0
	1c:		00000033		add x0 x0 x0
	20:		00c02303		lw x6 12 x0
	24:		00000033		add x0 x0 x0
	28:		00000033		add x0 x0 x0
	2c:		01002583		lw x11 16 x0
	30:		00730513		addi x10 x6 7
	34:		00000033		add x0 x0 x0
	38:		00000033		add x0 x0 x0
	3c:		00000033		add x0 x0 x0
	40:		00a5e5b3		or x11 x11 x10
	# end
```
will be translated to
```
147
2
32
0
19
3
48
0
... # and so on
```

## Code Samples

The two main objects in the simulator are the `CPU` and the `memory_driver`.
Since the bulk of the project was made as a class project, I cannot show my implementation of the entire program.
However, the interface of each class is as follows:

{% highlight cpp %}

class CPU {
public:
    CPU(uint8_t *inst_mem, uint32_t cache_type);
    ~CPU();

    void fetch(instruction &inst, uint32_t pc);
    void decode(instruction &inst);
    void execute(instruction &inst, int32_t &alu_result, uint32_t &read_data_2);
    void memory(instruction &inst, uint32_t &address_read_data, uint32_t &write_data_alu_result);
    void writeback(instruction &inst, uint32_t &read_data, uint32_t &alu_result, uint32_t &write_data);

    uint8_t  *get_inst_mem() const { return m_inst_mem; }
    uint8_t  *get_data_mem() const { return m_data_mem; }
    uint32_t *get_reg_file() const { return m_reg_file; }

    statistics m_stats;
private:
    uint8_t  *m_inst_mem;
    uint8_t  *m_data_mem;
    uint32_t *m_reg_file;
    memory_driver *m_mem_controller;
};

{% endhighlight %}

{% highlight cpp %}

class memory_driver {
public:
    memory_driver(int cache_type, uint8_t *mem);
    ~memory_driver();

    void _(uint32_t &data, bool is_read, uint32_t addr, statistics &stats);

private:
    void load_word();
    void store_word();
    void update(int current_lru, int set);
    void evict();
    bool search(bool is_read);

    int m_status;
    uint32_t m_data;
    uint32_t m_addr;
    uint8_t  m_type;
    uint8_t *m_mem;
    cache_set *m_cache;
    statistics m_stats;
};

{% endhighlight %}

`instruction` is a struct that contains all of the information each RISC-V instruction encodes as defined in the above link.
`statistics` is a struct that contains miscellaneous stats such as `clock`, `lw_miss_rate`, `sw_miss_rate`, etc.