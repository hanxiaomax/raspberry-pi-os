## Lesson 3: Interrupts (Raspberry PI OS)

From the lesson 1 we already know how to communicate with hardware. However, most of the times the pattern of communication is not that simple. Usually this pattern is asyncronos: we send some command to some device, but it doesn't respond imidiately. Instead it notifies us when command is complited. Those asyncronos notifications are called "interrupts" because they interrupts normal execution flow and forces processor to execute "interrupt handler" first and only then returns to the normal flow. 

There is one  device that is particulary usefull in operating system development - system timer. It is a device that can be configured to periodically interrupt a processor with some configurable frequency. One particular application of timer interrupts is implementation of process scheduling. A scheduler needs to measure how long each process has been executed and use this information to select next process to run. This measurement is based on timer interrupts.

We are going to talk about process scheduling in details in the next lesson, but for now our task would be to initializa system timer and implement interrupt handler. 

### Interrupts vs exceptions 

In ARM.v8 architecture interrups are part of a more general term: exception. There are 4 types of exceptions

* `Synchronous exception` - Exceptions of this type are always caused by a currenly executed instruction. For example, you can use `str` instruction to store some data at unexistent memory location. In this case a syncronos exception will be generated. Syncronos exceptions also can be used to generate a "software intrrupt" Software interrupt is actually a syncros exception that is generated on purpose by executing `svc` instruction. We will use this mechanizm in lesson 5 to implement system calls.
* `IRQ` - Those are normal interrupts. They are always asyncronos, which means that they have nothing to do with the currently executed instruction.  In contrast to syncronos exceptions, they are always generated not by a processor itself, but by external hardware.
* `FIQ` - Those are called "fast interrupts" and exist soely for the purtose of priorietizing exceptions. It is posible to configure some interrupts as "normal" and other as "fast". Fast interrupts will be signaled first and will be handled by a separate exception handler. Linux doesn't use fast interrupts and we also are not going to do so.
* `SError` - SError stands for "System Error". Like `IRQ` and `FIQ`, `SError` is asyncronos and is generated by external hardware. Unlike `IRQ` and `FIQ`, `SError` always indicates some error condition. [Here](https://community.arm.com/processors/f/discussions/3205/re-what-is-serror-detailed-explanation-is-required) you can find an examle explaining when `SError` can be generated.

### Exception vectors

Each exception type needs its own handler. Also separate handlers should be defined for each different execution state, in which exception is generated. There are 4 different execution states, that are interesting from exception handling standpoint. If we are working at EL1 those states can be defined as  following:

1. `EL1t` - Exception is taken from EL1 when stack pointer was shared with EL0  This happens when `SPSel` register holds the value `0` 
1. `EL1h` - Exception is taken from EL1 when dedicated stack pointer is alocated for EL1. This means that `SPsel` holds the value `1` and this is the mode that we are currently using.
1. `EL0_64` - Exception is taken from EL0 executing in 64 bit mode.
1. `EL0_32` - Exception is taken from EL0 executing in 32 bit mode.

In total we need to define 16 exception handlers (4 excetion types multiplied by 4 execution states) A special structure that holds addresses of all exception handlers is called "excetion vector table" of just "vector table". The structure of a vector table is defined in `Table D1-6 Vector offsets from vector table base address` at page 1430 of the [AArch64-Reference-Manual](https://developer.arm.com/docs/ddi0487/latest/arm-architecture-reference-manual-armv8-for-armv8-a-architecture-profile) You can think of vector table as an array of exception vectors, where each exception vector (or handler) is a continuas sequence of instructions responsible for handling exceptions. Accordingly to `Table D!-6` from `AArch64-Reference-Manual` each exception vector can be `0x80` bytes at maximum. This is not much, but nobody prevents us from jumping to some other memory location from excetion vector. 

I think all of this will be much cleaner by example, so now it is time to  see how excetion vectors are implmented in RPI-OS. Everything related to exception handling is defined in [entry.S](https://github.com/s-matyukevich/raspberry-pi-os/blob/master/src/lesson03/src/entry.S) and we are going to start examining it right now.

The first usefull macro is called `ventry` and is used to create entries in the vector table.

```
	.macro	ventry	label
	.align	7
	b	\label
	.endm
```

As you might infer from this definition we are not going to handle exceptions right inside exception vector, but insted we jump to a label that is provided for the macro as `label` argument. We need `.align 7` instruction because all exception vectors should be located at offset `0x80` bytes from each other. 

Vector table is defined [here](https://github.com/s-matyukevich/raspberry-pi-os/blob/master/src/lesson03/src/entry.S#L64) and it consist of 16 `ventry` definitions. For now we are only interested in handling `IRQ` from `EL1h` but still we need to define all 16 handlers. This is not because of some handware requirement, but rather because we want to see meaningfull error message in case if something goes wronrg. All handlers that should never be executed in normal flow have `invalid` postfix and uses [handle_invalid_entry](https://github.com/s-matyukevich/raspberry-pi-os/blob/master/src/lesson03/src/entry.S#L64) Let's take a look on how this makro is defined.

```
	.macro handle_invalid_entry type
	kernel_entry
	mov	x0, #\type
	mrs	x1, esr_el1
	mrs	x2, elr_el1
	bl	show_invalid_entry_message
	b	err_hang
	.endm
```

In the first line you can see that another macro is used - `kernel_entry`. This makro will be discussed  shortly.
Then we call [show_invalid_entry_message](https://github.com/s-matyukevich/raspberry-pi-os/blob/master/src/lesson03/src/irq.c#L34) and prepare 3 arguments for it. The first artument is exception type that can take one of [this](https://github.com/s-matyukevich/raspberry-pi-os/blob/master/src/lesson03/include/entry.h#L6) values.It basicly tells us what exactly exception handler has been executed.
Second parameter is the most important one and is called `ESR` which stands for Exception Syndrome Register. This argument is taken from `esr_el1` register wich is describe on page 1899 of `AArch64-Reference-Manual`. This register contains  detaild information about what causes an exception. 
Third argument is importand mostly in case of sycronos exceptions. It value is taken from already familiar to us `elr_el1` register, which contains the address of the instruction that had beed executed when exception was generated. For syncronos exceptions this is also the instruction that causes the exception.
After `show_invalid_entry_message`  function prints all this information to the screen we put processor in an infinute loop because there is not much else we can do.

### Saving register state

After exception handler finishes execution we wand all general purpose registers to have the same values they had before the exception was generated. If this is not the case, an interrupt, that has nothing to do with currently executing code, can influence the behaviour of this conde in unnpredicatable way. That's why the wirst this we must do after exception is generated is to save the processor state. This is done in the [kernel_entry](https://github.com/s-matyukevich/raspberry-pi-os/blob/master/src/lesson03/src/entry.S) marco. This macro is very simple - it just stores registers `x0 - x30` to the stack. There is also a coresponding  makro [kernel_exit](https://github.com/s-matyukevich/raspberry-pi-os/blob/master/src/lesson03/src/entry.S#L37) that is executed after each exception handler finishes execution. `kernel_exit` restores processor by copying bach values of `x0 - x30` registers. It also executes `eret` instruction witch returs us back to normal execution flow. By the way, general purpose registers are not the only thing that need to be saved before executing exception handler, but it is enough for our simple kernel for now. In later lessons we will add more functionality to the `kernel_entry` and `kernel_exit` macros.

### Setting vector table

Ok, now we have prepared vector table, but the processor don't know where it is located and therefor can't use it. In order for exception hanlig to work we must set `vbar_el1` (Vector Base Address Register) to the location of vector table. This is done [here](https://github.com/s-matyukevich/raspberry-pi-os/blob/master/src/lesson03/src/irq.S#L2)

```
.globl irq_vector_init
irq_vector_init:
	adr	x0, vectors		// load VBAR_EL1 with virtual
	msr	vbar_el1, x0		// vector table address
	ret
```

### Masking/unmasking interrupts

Another thing that we need to do is to unmask all types of interrupts. Let me explain what I mean by "unmasking" an interrupt. Sometimes there is a need to tel that particula pice of code must never be intercepted by an asyncronos interrupt. Imagine, for example, what heppens if an interrupt occures right in the middle of execution of `kernel_entry` macro? In this case processor state would be overriten and lost. That's why whenever exception handler is executed   processor automatically disables all types of interrupts. This is called "masking" and this also can be done manually if we need to do so. 

Many people mistakenly think that interrupts must be masked for the whole duration of an exception handler. This isn't true - it is perfectly legal to unmask interrupts after you saved processor state and it perfectly legal to have nested interrupts. We are not going to do this, but this is important information to keep in mind.

The [following two functions](https://github.com/s-matyukevich/raspberry-pi-os/blob/master/src/lesson03/src/irq.S#L7-L15) are responsible for masking and unmasking interrupts.

```
.globl enable_irq
enable_irq:
	msr    daifclr, #2 
	ret

.globl disable_irq
disable_irq:
	msr	daifset, #2
        ret
```

ARM processor state has 4 bits that are responsible for holding mask status for different types of interrupts. Those bits are defined as following

* `D` -  Masks debug exceptions. Those are special type of syncronos exceptions. For obvious reasons it is not posible to mask all syncronos exceptions, but it is convinient to have a separate flag that can mask debug exceptions.
* `A` - Masks `SErrors`. It is called `A` because `SErrors` sometimes are called asynchronous aborts. 
* `I` - Masks `IRQs`
* `F` - Masks `FIQs`

Now you can probably guess why registers that are responsible for changing interrupt mask status are called `daifclr` and `daifset`. Those registers set and clear interrupt mask status bits in this processor state.

The last thing you may wonder about is why do we use constant value `2` in both of the functions? This is because we only want to set and cleanr second (`I`) bit. 

### Configuring interrupt controller

Devices usually don't interrupt processor directly, instead they rely on interrupt controller to do the job. Interrupt controller can be used to enable/disable interrupts send by different devices. We also use interrupt controller to figure out what device generates an interrupt in an interrupt handler. Raspberry PI has its own interrupt handler that is described on page 109 of [BCM2835 ARM Peripherals manual](https://www.raspberrypi.org/documentation/hardware/raspberrypi/bcm2835/BCM2835-ARM-Peripherals.pdf) 

Rapberry PI interrupt controller has 3 registers that holds enabled/disabled status for all types of interrupts. For now we are only interested in timer interrupts, and those interrupts can be enabled using [ENABLE_IRQS_1](https://github.com/s-matyukevich/raspberry-pi-os/blob/master/src/lesson03/include/peripherals/irq.h#L10) register, which is described on page 116 of `BCM2835 ARM Peripherals manual`. On page 113 there is an ARM peripherial interrupt table. This table have 64 lines. Accordingly to the documentation interrupts are divided into 2 banks. First bank consist of interrupts `0 - 32`, each of those interrupts can be enabled or disabled by setting different bits of `ENABLE_IRQS_1` register. There is also a conresponsig register for the last 32 interrupts - `ENABLE_IRQS_2` and a register that controlls some common interrupts together with ARM local interrupts - `ENABLE_BASIC_IRQS` (We will talk about ARM local interrupts in the next chaper of this lesson). The manual though have a lot of mistakes and one of those is directly relevant to our discussion. Peripheral interrupt table should contain 4 interrupts from system timer at lines `0 - 3`. From reverse ingenering linux source code and reading other source (in particular [this one](http://embedded-xinu.readthedocs.io/en/latest/arm/rpi/BCM2835-System-Timer.html)) I was able to figure out that timer interrupts 0 and 2 are reserved and used by GPU and interrupts 1 and 3 can be used for any other purposes. So here is the [function](https://github.com/s-matyukevich/raspberry-pi-os/blob/master/src/lesson03/src/irq.c#L29) that enables IRQ number 1 from system timer.

```
void enable_interrupt_controller()
{
	put32(ENABLE_IRQS_1, SYSTEM_TIMER_IRQ_1);
}
```

### Generic IRQ handler

From out previous discussion you should remember that we have a single exception handler that is responsible for handling all `IRQs`. This handler is defined [herea(https://github.com/s-matyukevich/raspberry-pi-os/blob/master/src/lesson03/src/irq.c#L39)]

```
void handle_irq(void)
{
	unsigned int irq = get32(IRQ_PENDING_1);
	switch (irq) {
        case (SYSTEM_TIMER_IRQ_1):
			handle_timer_irq();
			break;
		default:
			printf("Inknown pending irq: %x\r\n", irq);
	}
}
```

In the handler we need a way to figure out what device was responsible for generating an interrupt. Interrupt controller can help us with this job: it has `IRQ_PENDING_1` register that holds interrupt status for interrupts `0 - 31`. Using this register we cah chech whether current interrupt was generated by timer or by any other device and call device specific interrupt handler. Note, that multiple interrupts can be pending at the same time. That's why each device specific interrup handler must acknolege that it completed handling interrupt and only after that interrupt pending bit will be cleared. Because of the same reason for a production ready OS you would probaly want to wrap switch construct in the generic interrupt handler in a loop: in this way you will be able to handle multiple interrupts at once. 

### Timer initialization

Raspberry Pi system timer is a very simple divice. It has a counter that increases its value by 1 after each clock tick. It also have 4 interrupt lines that connects it with an interrupt controller(so it can generate 4 different interrupts)  and 4 coresponsiding compare registers. When value of the counter becomes equal to the value stored in one of the compare registres - a coresponsding interrupt is generated. That's why befor we will be able to use sytem timer interrupts we need to initialize one of the compare registers with some non 0 value, the large the value is - the later an interrupt will be generated. This is done in [timer_init](https://github.com/s-matyukevich/raspberry-pi-os/blob/master/src/lesson03/src/timer.c#L8) function.

```
const unsigned int interval = 200000;
unsigned int curVal = 0;

void timer_init ( void )
{
    curVal = get32(TIMER_CLO);
    curVal += interval;
    put32(TIMER_C1, curVal);
}
```

The first line reads current counter value, second line increases it and third line sets compare register for interrupt 1 to the calculated value. 

### Handing timer interrupt

Finally we got to the timer interrupt handler. It is actually very simple.

```
void handle_timer_irq( void ) 
{
    curVal += interval;
    put32(TIMER_C1, curVal);
    put32(TIMER_CS, TIMER_CS_M1);
    printf("Timer iterrupt received\n\r");
}
```

Here we first update compare register, so that next interrupt will be generated after the same time interval. Next we acnolege an interrupt by writing 1 to 

### Conclusion

The last thing that might want to take a look at is the [kernel_main](https://github.com/s-matyukevich/raspberry-pi-os/blob/master/src/lesson03/src/kernel.c#L7) function where all previously discussed functionlity is orchestrated. After you compile and run the sample is should print "Timer iterrupt received" message afer each timer interrup is received. Please, try to do it by yourself and don't forget to carefully examine the code and experiment with it.