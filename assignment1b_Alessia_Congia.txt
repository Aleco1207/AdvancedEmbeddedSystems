/******************************************************************************
*
* Copyright (C) 2009 - 2014 Xilinx, Inc.  All rights reserved.
*
* Permission is hereby granted, free of charge, to any person obtaining a copy
* of this software and associated documentation files (the "Software"), to deal
* in the Software without restriction, including without limitation the rights
* to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
* copies of the Software, and to permit persons to whom the Software is
* furnished to do so, subject to the following conditions:
*
* The above copyright notice and this permission notice shall be included in
* all copies or substantial portions of the Software.
*
* Use of the Software is limited solely to applications:
* (a) running on a Xilinx device, or
* (b) that interact with a Xilinx device through a bus or interconnect.
*
* THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
* IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
* FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL
* XILINX  BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY,
* WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF
* OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
* SOFTWARE.
*
* Except as contained in this notice, the name of the Xilinx shall not be used
* in advertising or otherwise to promote the sale, use or other dealings in
* this Software without prior written authorization from Xilinx.
*
******************************************************************************/

/*
 * helloworld.c: simple test application
 *
 * This application configures UART 16550 to baud rate 9600.
 * PS7 UART (Zynq) is not initialized by this application, since
 * bootrom/bsp configures it to baud rate 115200
 *
 * ------------------------------------------------
 * | UART TYPE   BAUD RATE                        |
 * ------------------------------------------------
 *   uartns550   9600
 *   uartlite    Configurable only in HW design
 *   ps7_uart    115200 (configured by bootrom/bsp)
 */

#include <stdio.h>
#include "platform.h"
#include "xil_printf.h"

// I/O interfaces with respective addresses
volatile int * gpio_0_data = (volatile int*) 0x40000000; // LEDs
volatile int * gpio_1_data = (volatile int*) 0x40010000; // Switches
volatile int * gpio_2_data = (volatile int*) 0x40020000; // Button1

// AXI Interrupt Controller Register with respective addresses
volatile int * IER = (volatile int*) 0x41200008; // Interrupt Enable Register
volatile int * MER = (volatile int*) 0x4120001C; // Master Enable Register
volatile int * IISR = (volatile int*) 0x41200000; // Interrupt Status Register
volatile int * IIAR = (volatile int*) 0x4120000C; // Interrupt Acknowledge Register

// Registers' addresses for GPIO 1 (Switches, connected to INTC[0])
volatile int * GGIER_1 = (volatile int*) 0x4001011C; // Global GPIO Interrupt Enable Register
volatile int * GIER_1 = (volatile int*)	0x40010128; // GPIO Interrupt Enable Register
volatile int * GISR_1 = (volatile int*)	0x40010120; // GPIO Interrupt Status Register

// Registers' addresses for GPIO 2 (Button1, connected to INTC[1])
volatile int * GGIER_2 = (volatile int*) 0x4002011C; // Global GPIO Interrupt Enable Register
volatile int * GIER_2 = (volatile int*)	0x40020128; // GPIO Interrupt Enable Register
volatile int * GISR_2 = (volatile int*)	0x40020120; // GPIO Interrupt Status Register

// myISR() is the interrupt handler function
void myISR(void) __attribute__((interrupt_handler));

int main(void)
{

    // Set inputs
    *(volatile int*)(0x40010000 + 0x4) = 0xFFFFFFFF; // GPIO1 TRI
    *(volatile int*)(0x40020000 + 0x4) = 0xFFFFFFFF; // GPIO2 TRI

    // Enable device interrupts (GPIO)
	*GIER_1  = 0x1; // Enable interrupt for GPIO 1
	*GGIER_1 = 0x80000000; // Master enable for GPIO 1

	*GIER_2  = 0x1; // Enable interrupt for GPIO 2
	*GGIER_2 = 0x80000000; // Master enable for GPIO 2

	// Enable INTC lines
    *IER = 0x3; // 0b11 // enable two channels (11) -> IRQ0 and IRQ1 (use 0x2 if only GPIO2 is wired)
    *MER = 0x3; // 0b11 // ME | HIE

    microblaze_enable_interrupts(); // Enable processor interrupts

    // The CPU is on a listening state, provided by this infinite while loop.
    //In this way, any GPIO-triggered interrupt will stop this dormant state by executing the Interrupt Service Routine (ISR).
    while (1) {

    }
}

// Interrupt Service Routines
void myISR(void)
{
    unsigned p = *IISR;  // Snapshot
    if (p & 0b0001) { // if INTC[0]==1, the interrupt source is GPIO1
    	*GISR_1 = 0x1;    // clear device first
    	*IIAR   = 0x1;    // then ack INTC

    	if (*gpio_1_data & 0b1000) {                    //Check if the 4th bit is HIGH
    				*gpio_0_data = 0b0011;              //Then set LEDs representing 3
    	    	} else if (*gpio_1_data & 0b0100) {     //Check if the 3rd bit is HIGH
    	    		*gpio_0_data = 0b0010;              //Then set LEDs representing 2
    	    	} else if (*gpio_1_data & 0b0010) {     //Check if the 2nd bit is HIGH
    				*gpio_0_data = 0b0001;              //Then set LEDs representing 1
    	    	} else if (*gpio_1_data & 0b0001) {     //Check if the 1st bit is HIGH
    				*gpio_0_data = 0b0000;              //Then set LEDs representing 0
    	    	} else {
    	    		*gpio_0_data = 0b0000;              //Otherwise set LEDs representing 0
    	    	}

    }

    if ((p & 0b0010)) { // if INTC[1]==1, the interrupt source is GPIO2
           	if (*gpio_2_data) {
       			*GISR_2 = 0x1;
       			*IIAR   = 0x2;
       			// Inverts the current LED value
       			*gpio_0_data = ~(*gpio_0_data);

           	}
           }

}
