/*
 *Derek Chiu
 *400062312
 *Chiud5
 *2DP4 Final Project
 */
#include <hidef.h>      /* common defines and macros */
#include "derivative.h"      /* derivative-specific definitions */
#include "SCI.h"


void OutCRLF(void){
  SCI_OutChar(CR);
  SCI_OutChar(LF);
}

void setClk(void);
void delay1ms(unsigned int multiple);
  
unsigned short button = 0;
unsigned short val = 0;
unsigned short max = 0;
unsigned short min = 0;
unsigned short threshold = 0;
unsigned short flag = 0;
unsigned short postTotal = 0;
unsigned int beat = 0;
unsigned int count = 0;
unsigned int bpm = 0;
char lut[10] = {0x00,0x01,0x02,0x03,0x04,0x05,0x06,0x07,0x08,0x09};

void main(void) {
  setClk();  //set clock speed to 14MHz
  SCI_Init(57600); //set baudrate to 57600
  
 
//Setting up ports for output
  DDRJ = 0x01;          // Set pin D13 (on board LED) to an output
  PTJ = 0x00;           // Make sure the LED isn't on to begin with, since that would be confusing

  
//Timing for Interrupts
/*
 * The next six assignment statements configure the Timer Input Capture                                                   
 */           
  TSCR1 = 0x90;    //Timer System Control Register 1
                    // TSCR1[7] = TEN:  Timer Enable (0-disable, 1-enable)
                    // TSCR1[6] = TSWAI:  Timer runs during WAI (0-enable, 1-disable)
                    // TSCR1[5] = TSFRZ:  Timer runs during WAI (0-enable, 1-disable)
                    // TSCR1[4] = TFFCA:  Timer Fast Flag Clear All (0-normal 1-read/write clears interrupt flags)
                    // TSCR1[3] = PRT:  Precision Timer (0-legacy, 1-precision)
                    // TSCR1[2:0] not used

  TSCR2 = 0x00;    //Timer System Control Register 2
                    // TSCR2[7] = TOI: Timer Overflow Interrupt Enable (0-inhibited, 1-hardware irq when TOF=1)
                    // TSCR2[6:3] not used
                    // TSCR2[2:0] = Timer Prescaler Select: See Table22-12 of MC9S12G Family Reference Manual r1.25 (set for bus/1)
  
                    
  TIOS = 0xFE;     //Timer Input Capture or Output capture
                    //set TIC[0] and input (similar to DDR)
  PERT = 0x01;     //Enable Pull-Up resistor on TIC[0]

  TCTL3 = 0x00;    //TCTL3 & TCTL4 configure which edge(s) to capture
  TCTL4 = 0x02;    //Configured for falling edge on TIC[0]

/*
 * The next one assignment statement configures the Timer Interrupt Enable                                                   
 */           
   
  TIE = 0x01;      //Timer Interrupt Enable

/*
 * The next one assignment statement configures the ESDX to catch Interrupt Requests                                                   
 */           
  
	EnableInterrupts; //CodeWarrior's method of enabling interrupts
	
	
	ATDCTL1 = 0x27;   // Set resolution to 10 bits
	ATDCTL3 = 0x88;		// right justified, one sample per sequence
	ATDCTL4 = 0x06;		// prescaler = 6; ATD clock = 14MHz / (2 * (6 + 1)) == 1.0MHz
  ATDCTL5 = 0x27;		// continuous conversion on channel 7
  
 
  //Main Algorithm begins
    


  for(;;) {
    
    if(button == 1){
      val=ATDDR0;
      SCI_OutUDec(val);
      OutCRLF();
      if(count == 0){
        max = val;
        min = val;  
      } else if(count < 40){
        if(val > max){
          max = val;
        }
        if(val < min){
          min = val;
        }
      } else if(count == 40){
        threshold = (max+min)/2;
        max = val;
      } else {
        min = max;
        max = val;              
        if(flag == 0 && (min+max)/2 > (threshold)){
          flag = 1;
        }else if (flag ==1 && (min+max)/2 < (threshold)){
          flag = 0;
          beat = beat +1;
        }
      }
      if(count == 341){
      //change ports to ad for bcd output
        DDR0AD = 0x0F;
        DDR1AD = 0x0F;
        bpm = beat*4;
        if(bpm > 100){
          PTJ = 0x01;
          bpm = bpm-100;
        }else{
          PTJ = 0x00;
        }
        PT0AD = lut[bpm/10];
        bpm = bpm%10;
        PT1AD = lut[bpm];
        count = 41;
        beat = 0;
      //change back to adc 	
      	ATDCTL1 = 0x27;   // Set resolution to 10 bits
      	ATDCTL3 = 0x88;		// right justified, one sample per sequence
      	ATDCTL4 = 0x06;		// prescaler = 6; ATD clock = 14MHz / (2 * (6 + 1)) == 1.0MHz
        ATDCTL5 = 0x27;		// continuous conversion on channel 7
  
      }         
      delay1ms(50);
      count = count + 1;
    }else{
     count = 0;
     beat = 0;
     PTJ = 0x00;
     PT0AD = 0x00;
     PT1AD = 0x00; 
    }
  } /* loop forever */
  /* please make sure that you never leave main */
}


/*
 * This is the Interrupt Service Routine for TIC channel 0 (Code Warrior has predefined the name for you as "Vtimch0"                                                    
 */           
interrupt  VectorNumber_Vtimch0 void ISR_Vtimch0(void)
{
  unsigned int temp;
  
  if(button == 0){
      button = 1;
    } else {
      button = 0;
    }
     

  temp = TC0;       //Refer back to TFFCA, we enabled FastFlagClear, thus by reading the Timer Capture input we automatically clear the flag, allowing another TIC interrupt
}

   
void delay1ms(unsigned int multiple){
  int ix;/* enable timer and fast timer flag clear */
  TC2 = TCNT + 14000;
  for(ix = 0; ix < multiple; ix++) {
    while(!(TFLG1_C2F)); {
      TC2 += 14000;
    }
  } 
}
   


void setClk(void){ //Set E-Clock speed to 14 MHz
 
 CPMUREFDIV = 0x00;
 CPMUSYNR = 0x5A;
 CPMUPOSTDIV = 0x01;
 CPMUCLKS = 0x80;
 CPMUOSC = 0x00;
 
 while(!(CPMUFLG & 0x08));
 
}