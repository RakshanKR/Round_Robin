#include <stdint.h>
#include <stdio.h>
#include <stdbool.h>
#include "TM4C123.h"

#define DS_PIN (1U << 2) // PE2
#define SHCP_PIN (1U << 3) // PE3
#define STCP_PIN (1U << 4) // PE4
#define SYSCTL_RCGCGPIO_R4 0x400FE608 | (1 << 4)
#define DISPLAY_SELECT_PINS (1U << 5) | (1U << 6) | (1U << 7) | (1U << 8) // PE5, PE6, PE7, PE8

void delay(uint32_t time);
void shiftOut(uint8_t data);
uint8_t convertTo7Segment(uint8_t number);

void delay1(int value) {
    for (int i = 0; i <= value; i++) {
        for (int j = 0; j <= value; j++)
            ;
    }
}

void shiftOut(uint8_t data) {
    for (int i = 0; i < 8; i++) {
        if (data & 0x80) {
            GPIOE->DATA |= DS_PIN;
        } else {
            GPIOE->DATA &= ~DS_PIN;
        }
        GPIOE->DATA |= SHCP_PIN;
        delay1(10);
        GPIOE->DATA &= ~SHCP_PIN;
        delay1(10);
        data <<= 1;
    }
    GPIOE->DATA |= STCP_PIN;
    delay1(0x10);
    GPIOE->DATA &= ~STCP_PIN;
    delay1(0x10);
}

uint8_t convertTo7Segment(uint8_t number) {
    static const uint8_t segmentMap[10] = {
        0x3f,  // 0
        0x07,  // 1
        0x5b,  // 2
        0x4f,  // 3
        0x66,  // 4
        0x6d,  // 5
        0x7d,  // 6
        0x07,  // 7
        0x7f,  // 8
        0x4f   // 9
    };
    if (number < 10) {
        return segmentMap[number];
    } else {
        return 0x3f;
    }
}

int main() {
    int burst_times[] = {4, 0, 2, 4, 1};
    int process_count = 5;
    int completed_processes = 0;
    int current_process = 0;
    int quantum_time = 2;
int display_value =0;

    // Initialize GPIOE
    SYSCTL->RCGCGPIO |= SYSCTL_RCGCGPIO_R4;
    while (!(SYSCTL->PRGPIO & SYSCTL_RCGCGPIO_R4)) { };

    // Set PE2, PE3, and PE4 as output for DS, SHCP, and STCP
    GPIOE->DIR |= DS_PIN | SHCP_PIN | STCP_PIN;
    GPIOE->DEN |= DS_PIN | SHCP_PIN | STCP_PIN;

    // Set PE5, PE6, PE7, and PE8 as output for display select
    GPIOE->DIR |= DISPLAY_SELECT_PINS;
    GPIOE->DEN |= DISPLAY_SELECT_PINS;

    round_robin:
    while (completed_processes != process_count) {
        if (burst_times[current_process] >= quantum_time) {
            burst_times[current_process] -= quantum_time;
  display_value = burst_times[current_process];
for (int i = 0; i < 4; i++) {
            GPIOE->DATA &= ~(DISPLAY_SELECT_PINS);
            GPIOE->DATA |= (1U << (5 + i)); // Select the current display
            shiftOut(convertTo7Segment(display_value % 10));
            display_value /= 10;
            delay1(2000); // Display for 1ms
        }
        } else {
            if (burst_times[current_process] == 0) {
                completed_processes++;
            }
            if (burst_times[current_process] < quantum_time) {
                burst_times[current_process] = 0;
            }
        }

        // Display the value on the 4 displays
       
       

        if (current_process == process_count - 1) {
            current_process = 0;
        } else {
            current_process++;
        }
    }

    completed_processes = 0;
    for (int i = 0; i < process_count; i++) {
        if (burst_times[i] < quantum_time) {
            completed_processes++;
        }
    }

    if (completed_processes <= process_count) {
        completed_processes = 0;
       goto round_robin;
    }

    return 0;
}
