---
title: "writel and readl"
date: 2025-12-25
categories:
  - linux-kernel
tags:
  - wirtel
  - readl
  - kernel
---

`writel` and `readl` are **Linux kernel functions for 32-bit memory-mapped I/O (MMIO) operations** on device registers. They ensure proper ordering, handle endianness, and work with volatile memory semantics.

## 📋 Function Signatures & Basic Usage

```c
#include <linux/io.h>

/* Write 32-bit value to MMIO address */
void writel(u32 value, volatile void __iomem *addr);

/* Read 32-bit value from MMIO address */
u32 readl(const volatile void __iomem *addr);
```

### Basic Example
```c
/* Writing to a PCIe device control register */
void enable_device(struct pcie_device *dev)
{
    u32 ctrl_reg;
    
    /* Read current control register */
    ctrl_reg = readl(dev->regs + CTRL_REG_OFFSET);
    
    /* Set enable bit (bit 0) */
    ctrl_reg |= 0x1;
    
    /* Write back to device */
    writel(ctrl_reg, dev->regs + CTRL_REG_OFFSET);
}
```

## 🔍 Architecture-Specific Implementations

### **x86/x86_64 Implementation**
```c
/* From arch/x86/include/asm/io.h */

/* readl - Read 32-bit from MMIO */
static inline u32 readl(const volatile void __iomem *addr)
{
    u32 val;
    
    /* "volatile" prevents compiler optimization */
    asm volatile("movl %1, %0"
                 : "=r" (val)          /* Output: val = result */
                 : "m" (*(volatile u32 __force *)addr)  /* Input: memory at addr */
                 : /* No clobbers */);
    return val;
}

/* writel - Write 32-bit to MMIO */
static inline void writel(u32 val, volatile void __iomem *addr)
{
    /* Force write to memory, not optimized away */
    asm volatile("movl %0, %1"
                 : /* No outputs */
                 : "r" (val),          /* Input: value to write */
                   "m" (*(volatile u32 __force *)addr)  /* Destination */
                 : /* No clobbers */);
}
```

### **ARM/ARM64 Implementation**
```c
/* From arch/arm/include/asm/io.h */

#define readl(c) ({                               \
    u32 __v = __raw_readl(__io_pci(c));           \
    __iormb(); /* Read memory barrier */          \
    __v;                                          \
})

#define writel(v, c) ({                           \
    __iowmb(); /* Write memory barrier */         \
    __raw_writel(v, __io_pci(c));                 \
})

/* __raw_readl/writel - No barriers */
#define __raw_readl(a)        (*(volatile u32 __force *)(a))
#define __raw_writel(v, a)    (*(volatile u32 __force *)(a) = (v))
```

## 🎯 Key Design Concepts

### **1. Volatile Keyword**
```c
/* Without volatile (WRONG) */
u32 *reg = dev->regs;
*reg = 0x12345678;  /* Compiler might optimize this away */

/* With volatile (CORRECT) */
volatile u32 *reg = dev->regs;
*reg = 0x12345678;  /* Compiler preserves this write */
```

### **2. Memory Barriers**
| Function | Barrier Type | Purpose |
|----------|--------------|---------|
| `writel()` | **Write barrier** | Ensures previous writes complete before this write |
| `readl()` | **Read barrier** | Ensures this read completes before subsequent reads |
| `writel_relaxed()` | No barrier | When ordering doesn't matter (performance) |

### **3. Endianness Handling**
```c
/* PCIe devices are typically little-endian */
#ifdef __LITTLE_ENDIAN
    /* readl/writel work directly */
#else
    /* Big-endian hosts need byte swapping */
    static inline u32 readl(const volatile void __iomem *addr)
    {
        return swab32(__raw_readl(addr));
    }
#endif
```

## 📊 Complete Implementation Examples

### **Example 1: Device Reset Sequence**
```c
#define DEVICE_RESET_REG     0x00
#define DEVICE_STATUS_REG    0x04
#define RESET_BIT            (1 << 0)
#define RESET_COMPLETE_BIT   (1 << 1)

int reset_device(struct my_device *dev)
{
    u32 status;
    int timeout = 1000;
    
    /* 1. Assert reset (set bit 0) */
    writel(RESET_BIT, dev->regs + DEVICE_RESET_REG);
    
    /* 2. Wait for reset to complete */
    do {
        /* readl includes memory barrier */
        status = readl(dev->regs + DEVICE_STATUS_REG);
        
        if (status & RESET_COMPLETE_BIT)
            break;
            
        udelay(10);  /* 10 microsecond delay */
    } while (--timeout > 0);
    
    if (!timeout) {
        dev_err(dev->device, "Reset timeout, status: 0x%08x\n", status);
        return -ETIMEDOUT;
    }
    
    /* 3. Deassert reset (clear bit 0) */
    writel(0, dev->regs + DEVICE_RESET_REG);
    
    return 0;
}
```

### **Example 2: Register Polling with Timeout**
```c
#define POLL_INTERVAL_US     10  /* 10 microseconds */
#define POLL_TIMEOUT_US      100000  /* 100 milliseconds */

/* Generic polling function */
u32 poll_register(void __iomem *reg, u32 mask, u32 expected_val)
{
    u32 val;
    unsigned long timeout = jiffies + usecs_to_jiffies(POLL_TIMEOUT_US);
    
    do {
        val = readl(reg);
        
        if ((val & mask) == expected_val)
            return val;
            
        /* Wait before next read */
        if (time_after(jiffies, timeout))
            break;
            
        udelay(POLL_INTERVAL_US);
    } while (1);
    
    /* Timeout - return current value */
    return val;
}
```

### **Example 3: Batch Register Operations**
```c
/* Optimized: Write multiple registers */
void configure_device_bulk(struct device_regs *regs)
{
    /* Use writel for critical timing */
    writel(0x00000001, regs->ctrl_reg);    /* Enable */
    writel(0x0000ABCD, regs->config_reg);  /* Configuration */
    writel(0x00010000, regs->dma_addr);    /* DMA address */
    writel(0x00001000, regs->dma_len);     /* DMA length */
    
    /* Start operation */
    writel(0x00000002, regs->command_reg);
}

/* Optimized: Read multiple registers */
void read_device_status(struct device_regs *regs, u32 *status)
{
    /* readl ensures proper ordering */
    status[0] = readl(regs->status_reg1);
    status[1] = readl(regs->status_reg2);
    status[2] = readl(regs->error_reg);
    status[3] = readl(regs->fifo_count);
}
```

## ⚠️ Critical Implementation Details

### **1. Proper Alignment**
```c
/* WRONG: Unaligned access */
u16 read_half_register(void __iomem *reg)
{
    /* May cause alignment fault on some architectures */
    return *(volatile u16 *)reg;
}

/* CORRECT: Use appropriate accessor */
u16 readw(const volatile void __iomem *addr);  /* For 16-bit */
u32 readl(const volatile void __iomem *addr);  /* For 32-bit */
u64 readq(const volatile void __iomem *addr);  /* For 64-bit */
```

### **2. Memory Space Detection**
```c
/* Determine if address is MMIO or PORT I/O */
void access_device_registers(struct pci_dev *pdev)
{
    void __iomem *regs;
    
    /* Map BAR0 as MMIO */
    regs = pci_iomap(pdev, 0, pci_resource_len(pdev, 0));
    
    if (regs) {
        /* Use readl/writel for MMIO */
        writel(0x12345678, regs);
        u32 val = readl(regs);
    } else {
        /* Fall back to port I/O (x86 specific) */
        outl(0x12345678, pci_resource_start(pdev, 0));
        u32 val = inl(pci_resource_start(pdev, 0));
    }
}
```

### **3. Relaxed Variants for Performance**
```c
/* When ordering doesn't matter */
void configure_buffers(struct device *dev)
{
    /* These can be reordered by CPU for better performance */
    writel_relaxed(dev->dma_addr_lo, dev->regs + DMA_ADDR_LO);
    writel_relaxed(dev->dma_addr_hi, dev->regs + DMA_ADDR_HI);
    writel_relaxed(dev->dma_len, dev->regs + DMA_LENGTH);
    
    /* Memory barrier before starting DMA */
    wmb();
    
    /* This write must be ordered */
    writel(DMA_START, dev->regs + DMA_CTRL);
}
```

## 🔧 Debugging & Tracing

### **1. Adding Trace Points**
```c
/* In driver source */
#define DEBUG_REG_ACCESS 1

#ifdef DEBUG_REG_ACCESS
#define trace_write(reg, val) \
    pr_debug("WRITE: reg=%p, val=0x%08x\n", reg, val)
#define trace_read(reg, val) \
    pr_debug("READ: reg=%p, val=0x%08x\n", reg, val)
#else
#define trace_write(reg, val) do {} while (0)
#define trace_read(reg, val) do {} while (0)
#endif

/* Wrapper functions */
static inline void debug_writel(u32 val, void __iomem *reg)
{
    trace_write(reg, val);
    writel(val, reg);
}

static inline u32 debug_readl(void __iomem *reg)
{
    u32 val = readl(reg);
    trace_read(reg, val);
    return val;
}
```

### **2. Checking for Null Pointers**
```c
/* Safe wrapper with validation */
int safe_writel(struct device *dev, u32 val, u32 offset)
{
    if (!dev || !dev->regs) {
        dev_err(dev, "Device not initialized\n");
        return -ENODEV;
    }
    
    if (offset >= dev->regs_size) {
        dev_err(dev, "Offset 0x%x out of bounds (size=0x%x)\n",
                offset, dev->regs_size);
        return -EINVAL;
    }
    
    writel(val, dev->regs + offset);
    return 0;
}
```

## 📊 Performance Considerations

### **Benchmark: Relaxed vs Normal Access**
```c
/* Measure performance difference */
void benchmark_reg_access(void __iomem *reg)
{
    int i;
    u64 start, end;
    
    /* Test writel */
    start = ktime_get_ns();
    for (i = 0; i < 1000; i++)
        writel(i, reg);
    end = ktime_get_ns();
    pr_info("writel: %llu ns per write\n", (end - start) / 1000);
    
    /* Test writel_relaxed */
    start = ktime_get_ns();
    for (i = 0; i < 1000; i++)
        writel_relaxed(i, reg);
    end = ktime_get_ns();
    pr_info("writel_relaxed: %llu ns per write\n", (end - start) / 1000);
}
```

## 🎯 Summary of Key Points

| Aspect | `writel` | `readl` |
|--------|----------|---------|
| **Volatile** | ✅ Ensures write happens | ✅ Ensures read happens |
| **Barriers** | Includes write barrier | Includes read barrier |
| **Endianness** | Handles byte swapping | Handles byte swapping |
| **Alignment** | Requires 32-bit aligned address | Requires 32-bit aligned address |
| **Performance** | Slower (has barriers) | Slower (has barriers) |
| **Use Case** | Device control registers | Device status registers |

**Common Mistakes to Avoid:**
1. **Never** dereference `__iomem` pointers directly (use `readl`/`writel`)
2. **Always** use the correct size accessor (`readb`, `readw`, `readl`, `readq`)
3. **Consider** using relaxed versions in performance-critical paths
4. **Validate** offsets before accessing registers

If you need to see how these functions interact with specific hardware like DOE mailboxes or DMA controllers, I can provide more targeted examples.
