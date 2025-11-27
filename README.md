Bien, anteriormente comparti la version con PIC ( https://github.com/ETI-PATAGONIA-AR/Controlador-L-gico-Programable-de-Tiempo-V1 ) y hace un tiempo, arranque a redactar un libro para 
autodidactas de programación de Arduino desde cero, y llegue a un punto en el que se me habían acabado las ideas de que proyectos subir a modo de ejemplo, así que tome una carpeta vieja
de proyectos en PBP y me tome el trabajo de sacarme HUMO de la cabeza para poder pasar de un lenguaje a otro todo.... Sin los Gosub-Return y el modo en que trabaja el C a diferencia del 
Basic, la verdad que me llevo un tiempo acostumbrarme a tomar determinados hábitos para poder tener las mismas funciones... En este ejemplo simplificado, muestro como podemos hacer un 
relay controlado por tiempo implementando la librería "TimerONE.h", y la navegación de menú o funciones por medio del empleo de flags para indicar a donde ir, donde quedarse o a donde 
volver cuando tuve que saltar....

```
////////////////////////////////////////////////////////////////////////////////
////////////////////////////////////////////////////////////////////////////////
#include "TimerOne.h"
#include "H_Config.h"
#include <LiquidCrystal_I2C.h>
LiquidCrystal_I2C lcd(0x27,20,4);
#include <EEPROM.h>
#include <Wire.h>

#define START_button 3   // boton inicio
#define STOP_button  4   // boton parada
#define CLEAR_button 5   // boton borrar
#define MAS_button   6    // boton mas
#define MENOS_button 7    // boton menos
#define ENTER_button 8   // entrada disparo auxiliar

//Definimos los puertos afectados a los Relay de control
int RELAY_1 = 10;

//Definimos las variables globales del programa
int banderaPROG;
int bandera_MPROG;
int banderaPROG_R1=0;
int bandera=0;          //variable auxiliar
int BANDERA_R1=0;
int BANDERA_R1est=0;
int BANDERA_PARADA_R1 = 0;
int TICKS;             //variable auxiliar para los for-next
int i;                     //variable auxiliar para los for-next
int setHOUR_R1=0;       // variable set HORAS programadas
int setMINUTE_R1=0;      // variable set minutos programadas
int setSECOND_R1=0;
int AsetHOUR_R1=0;       // variable set HORAS programadas
int AsetMINUTE_R1=0;     // variable set minutos programadas
int AsetSECOND_R1=0;
iint HOUR = 0;         //variable HORAS cronometro
int MINUTE = 0;       //variable Minutos cronometro
int SECOND= 0;       //variable Segundos cronometro

//Variables afectadas al MENU
int menu = 1;
int modo = 0;
int listaMENU=0;
int listaMENU2=0;
int mode=0;

////////////////////////////////////////////////////////////////////////////////
////////////////////////////////////////////////////////////////////////////////
void setup()
    {
    Timer1.initialize(1000000); // 1 segundo
    Timer1.attachInterrupt(CONTADOR); // LLama a la funcion cuando se produce la interrupcion por timer 
  
    setHOUR_R1   =0;  //valor inicial de la variable
    setMINUTE_R1 =0;  //valor inicial de la variable
    setSECOND_R1 =0;  //valor inicial de la variable
    
    // Configuramos los puertos de ENTRADA
    pinMode(START_button, INPUT_PULLUP);  // Pin digital como entrada con conexión PULL-UP interna
    pinMode(STOP_button, INPUT_PULLUP);   // Pin digital como entrada con conexión PULL-UP interna
    pinMode(CLEAR_button, INPUT_PULLUP);  // Pin digital como entrada con conexión PULL-UP interna
    pinMode(MAS_button, INPUT_PULLUP);    // Pin digital como entrada con conexión PULL-UP interna
    pinMode(MENOS_button, INPUT_PULLUP);  // Pin digital como entrada con conexión PULL-UP interna
    pinMode(ENTER_button, INPUT_PULLUP);  // Pin digital como entrada con conexión PULL-UP interna
 
    // Configuramos los puertos de SALIDA
    pinMode(RELAY_1, OUTPUT);
  
    // Configuramos el estado de los puertos de SALIDA en el momento de arranque
    digitalWrite(RELAY_1,LOW);    //Poner en bajo el relay (tiene logica negativa)

    //Arrancamos con la pantalla de INICIO
    lcd.init();
    lcd.clear();
    lcd.backlight();
    lcd.setCursor(0,0);
    lcd.print("--------------------");
    lcd.setCursor(0,1);
    lcd.print("Tempo PROGRAMABLE Vx");
    lcd.setCursor(0,2);
    lcd.print("       ETI Patagonia");
    lcd.setCursor(0,3);
    lcd.print("--------------------");
    delay(900);
    lcd.clear();
    leerEEPROM();
    }
////////////////////////////////////////////////////////////////////////////////
////////////////////////////////////////////////////////////////////////////////

void loop()
    {
    modo = 0;
    digitalWrite(RELAY_1,LOW);
    lcd.setCursor(0,0);
    lcd.print("-MENU---------------");
    lcd.setCursor(0,1);
    lcd.print("INICIO     ->(ENTER)");
    lcd.setCursor(0,2);
    lcd.print("CONFIG     ->(STOP!)");
    lcd.setCursor(0,3);
    lcd.print("--------------------");
    Timer1.start();
    if((digitalRead (ENTER_button) == 0) || (digitalRead(START_button) == 0))
       {
       while (digitalRead(ENTER_button) == 0) { } // espera a que se suelte el pulsador
       leerEEPROM();
        lcd.clear();
        modo = 3;
        lcd.clear();
        Timer1.start();               // Habilitamos el timer1
        R_TempoON();
       }
  
     if(digitalRead(START_button) == 0)
       {
       while (digitalRead(START_button) == 0) { } // espera a que se suelte el pulsador
        leerEEPROM();
        lcd.clear();
        modo = 3;
        SECOND=0;
        lcd.clear();
        Timer1.start();    // Habilitamos el timer1 
        R_TempoON();
       }
    
      if(digitalRead (STOP_button) == 0)
        {
         while (digitalRead(STOP_button) == 0) { } // espera a que se suelte el pulsador
         leerEEPROM();
         modo = 10;
         lcd.clear();
          progTEMPO_R1();
         }
     }
////////////////////////////////////////////////////////////////////////////////
//************* INICIO SUBPROGRAMA PARA CONFIGURAR TIEMPO R1****************
////////////////////////////////////////////////////////////////////////////////
void progTEMPO_R1()
    {
    lcd.clear();
    CONFIG_R1_SEG();
    }

////////////////////////////////////////////////////////////////////////////////
//**************** sub programa para configurar el tiempo ******************
////////////////////////////////////////////////////////////////////////////////
void CONFIG_R1_SEG()
    {
     modo=10;
     lcd.setCursor(0,0);
     lcd.print("-Config-R1----------");
     lcd.setCursor(0,1);
     lcd.print("            >>SEG<< ");
     lcd.setCursor(0,2);
     lcd.print("TIEMPO: ");
     if(setHOUR_R1<=9){lcd.print("0");}
     lcd.print(setHOUR_R1);
     lcd.print(":");
     if(setMINUTE_R1<=9){lcd.print("0");}
     lcd.print(setMINUTE_R1);
     lcd.print(":");
     if(setSECOND_R1<=9){lcd.print("0");}
     lcd.print(setSECOND_R1);
     lcd.print("   ");
  
     if(digitalRead (MAS_button) == 0)
       {
        setSECOND_R1++;
        if(setSECOND_R1>59){setSECOND_R1=0;}
        delay(200);
       }
     if(digitalRead (MENOS_button) == 0)
       {
        setSECOND_R1--;
        if(setSECOND_R1<0){setSECOND_R1=59;}
        delay(200);
       }
     if(digitalRead (ENTER_button) == 0)
       {
        while (digitalRead(ENTER_button) == 0) { } // espera a que se suelte el pulsador
        EEPROM.write(2, setSECOND_R1);
        BANDERA_R1 = 1;
        modo=11;
        setOK();
        CONFIG_R1_MIN();   
       }
     if(digitalRead (STOP_button) == 0)
       {
        while (digitalRead(STOP_button) == 0) { } // espera a que se suelte el pulsador
        EEPROM.write(2, setSECOND_R1);
        modo=0; 
        lcd.clear(); 
        loop();
        }
      if(digitalRead (CLEAR_button) == 0)
        {
         while (digitalRead(CLEAR_button) == 0) { } // espera a que se suelte el pulsador
         setSECOND_R1 = 0;
        }           
      if(modo==10)
        {
         CONFIG_R1_SEG();
        } 
     }   
////////////////////////////////////////////////////////////////////////////////
////////////////////////////////////////////////////////////////////////////////
void CONFIG_R1_MIN()
     {
     modo=11;
     lcd.setCursor(0,0);
     lcd.print("-Config-R1----------");
     lcd.setCursor(0,1);
     lcd.print("         >>MIN<<    ");
     lcd.setCursor(0,2);
     lcd.print("TIEMPO: ");
     if(setHOUR_R1<=9){lcd.print("0");}
     lcd.print(setHOUR_R1);
     lcd.print(":");
     if(setMINUTE_R1<=9){lcd.print("0");}
     lcd.print(setMINUTE_R1);
     lcd.print(":");
     if(setSECOND_R1<=9){lcd.print("0");}
     lcd.print(setSECOND_R1);
     lcd.print("   ");
  
     if(digitalRead (MAS_button) == 0)
       {
        setMINUTE_R1++;
        if(setMINUTE_R1>59){setMINUTE_R1=0;}
        delay(200);
       }
     if(digitalRead (MENOS_button) == 0)
       {
        setMINUTE_R1--;
        if(setMINUTE_R1<0){setMINUTE_R1=59;}
        delay(200);
       }
     if(digitalRead (ENTER_button) == 0)
       {
        while (digitalRead(ENTER_button) == 0) { } // espera a que se suelte el pulsador
        EEPROM.write(3, setMINUTE_R1);
        BANDERA_R1 = 1;
        modo=12;
        setOK();
        CONFIG_R1_HOR();   
       }
     if(digitalRead (STOP_button) == 0)
       {
        while (digitalRead(STOP_button) == 0) { } // espera a que se suelte el pulsador
        EEPROM.write(3, setMINUTE_R1);
        modo=0; 
        lcd.clear(); 
        loop();
        }
      if(digitalRead (CLEAR_button) == 0)
        {
         while (digitalRead(CLEAR_button) == 0) { } // espera a que se suelte el pulsador
         setMINUTE_R1 = 0;
        }           
      if(modo==11)
        {
         CONFIG_R1_MIN();
        }
     }
////////////////////////////////////////////////////////////////////////////////
////////////////////////////////////////////////////////////////////////////////
void CONFIG_R1_HOR()
    {
     modo=12;
     lcd.setCursor(0,0);
     lcd.print("-Config-R1----------");
     lcd.setCursor(0,1);
     lcd.print("      >>HOR<<       ");
     lcd.setCursor(0,2);
     lcd.print("TIEMPO: ");
     if(setHOUR_R1<=9){lcd.print("0");}
     lcd.print(setHOUR_R1);
     lcd.print(":");
     if(setMINUTE_R1<=9){lcd.print("0");}
     lcd.print(setMINUTE_R1);
     lcd.print(":");
     if(setSECOND_R1<=9){lcd.print("0");}
     lcd.print(setSECOND_R1);
     lcd.print("   ");
  
    if(digitalRead (MAS_button) == 0)
       {
        setHOUR_R1++;
        if(setHOUR_R1>98){setHOUR_R1=0;}
        delay(200);
       }

     if(digitalRead (MENOS_button) == 0)
       {
        setHOUR_R1--;
        if(setHOUR_R1<0){setHOUR_R1=98;}
        delay(200);
       }

     if(digitalRead (ENTER_button) == 0)
       {
        lcd.clear();
        while (digitalRead(ENTER_button) == 0) { } // espera a que se suelte el pulsador
        EEPROM.write(4, setHOUR_R1);
        BANDERA_R1 = 1;
        setOK();
        modo=1;
        banderaPROG_R1 = 1;
        menuPROG();   
       }

     if(digitalRead (STOP_button) == 0)
       {
        while (digitalRead(STOP_button) == 0) { } // espera a que se suelte el pulsador
        EEPROM.write(4, setHOUR_R1);
        modo=0; 
        lcd.clear(); 
        loop();
        }

      if(digitalRead (CLEAR_button) == 0)
        {
         while (digitalRead(CLEAR_button) == 0) { } // espera a que se suelte el pulsador
         setHOUR_R1 = 0;
        }           
      if(modo==12)
        {
         CONFIG_R1_HOR();
        }
    }
//////////////////////////////////////////////////////////////////
//////////////////////////////////////////////////////////////////
void R_TempoON()
    {
     banderaPROG=1;
     modo = 3;
     lcd.clear();
     leerEEPROM();
     INI_TEMPO();
     SECOND=0;
    }
////////////////////////////////////////////////////////////////////////////////
////////////////////////////////////////////////////////////////////////////////
void INI_TEMPO()
    {
      lcd.setCursor(0,0);
     lcd.print("--------------------");
     verDISPLAY();
  
      if (BANDERA_R1 == 1)
        {
         digitalWrite(RELAY_1,HIGH);    //Poner en alto el relay (tiene logica negativa)
         BANDERA_R1est=1;
        }
  
    if (SECOND == 60)        //; consigna: si variable segundos es = a 60, entonces
       {
        SECOND = 0;          //ponemos variable segundos en 0
        MINUTE ++;
       }
     if (MINUTE == 60)        //consigna dentro de consigna, si minuto es = 60 entonces
        {MINUTE = 0;        //ponemos variable minutos en 0
        HOUR ++;
        }   
     lcd.setCursor(0,0);
     lcd.print("--------------------");
     verDISPLAY();
      
      if ((setHOUR_R1 == HOUR) && (setMINUTE_R1 == MINUTE) && (setSECOND_R1 == SECOND))     //RELAY1
        {
         digitalWrite(RELAY_1,LOW);    //Poner en bajo el relay (tiene logica negativa)
         BANDERA_R1est=0;
         BANDERA_R1=0;
        }

     if (BANDERA_R1==0)
       {
        SECOND = 0;
        MINUTE = 0;
        HOUR = 0;
        leerEEPROM();
        delay(1500);
        lcd.clear();
        lcd.setCursor(0,0);
        lcd.print("--------------------");
        lcd.setCursor(0,1);
        lcd.print("  TIEMPO  PROGRAMA  ");
        lcd.setCursor(0,2);
        lcd.print("      TERMINADO     ");
        lcd.setCursor(0,3);
        lcd.print("--------------------");
         delay(2500);
         lcd.clear();
        BANDERA_R1=1;
        modo=0;
       loop();
       }
     if(digitalRead (STOP_button) == 0)
       {
        BANDERA_PARADA_R1 = 1;   // Flag para saber de donde me fui, y donde retornar
        Timer1.stop();                         // Paramos el timer para que no siga sumando tiempo
        PARADA();
       }
     if(modo==3)
        {
        INI_TEMPO();
        }
    }   

////////////////////////////////////////////////////////////////////////////////
//****************** sub programa para el cronometro ***********************
////////////////////////////////////////////////////////////////////////////////
void CONTADOR()
{
 SECOND ++;  ;  //incrementar 1 en variable segundo
}
void CONTADOR_MIN()
    {
    MINUTE ++;  //incrementamos 1 en variable minuto
    } 
void CONTADOR_HOR()
   {
    HOUR ++;   //incrementamos 1 en variable hora
   }
////////////////////////////////////////////////////////////////////////////////
////////////////////////////////////////////////////////////////////////////////
void verDISPLAY()
    {
     lcd.setCursor(0,1);
     lcd.print("R1:");lcd.print(BANDERA_R1est);
     lcd.setCursor(0,2);
     lcd.print("Tiempo: ");
     if(HOUR<=9){lcd.print("0");};lcd.print(HOUR);
     lcd.print(":");
     if(MINUTE<=9){lcd.print("0");};lcd.print(MINUTE);
     lcd.print(":");
     if(SECOND<=9){lcd.print("0");};lcd.print(SECOND);
     lcd.setCursor(0,3);
     lcd.print("--------------------");
    }
////////////////////////////////////////////////////////////////////////////////
////////////////////////////////////////////////////////////////////////////////
    void GrabarEEPROM()
    {
     EEPROM.write(2,setSECOND_R1);
     EEPROM.write(3,setMINUTE_R1);
     EEPROM.write(4,setHOUR_R1);
     EEPROM.write(14,bandera_MPROG);
    
    AsetHOUR_R1=setHOUR_R1;
    AsetMINUTE_R1=setMINUTE_R1;
    AsetSECOND_R1=setSECOND_R1;
   }
////////////////////////////////////////////////////////////////////////////////
////////////////////////////////////////////////////////////////////////////////
    void leerEEPROM()
        {
    setSECOND_R1 =  EEPROM.read(2);
    setMINUTE_R1 =  EEPROM.read(3);
    setHOUR_R1   =  EEPROM.read(4);
    bandera_MPROG = EEPROM.read(14);
     AsetHOUR_R1=setHOUR_R1;
    AsetMINUTE_R1=setMINUTE_R1;
    AsetSECOND_R1=setSECOND_R1;
   }
////////////////////////////////////////////////////////////////////////////////
//********************** sub programa para la pausa ************************
////////////////////////////////////////////////////////////////////////////////
void PARADA()
    {
      modo=70;
     Timer1.stop();
     digitalWrite(RELAY_1,LOW);    //Poner en bajo el relay
     BANDERA_R1 == 0;
     BANDERA_R1est=0;
     lcd.clear();
     for(TICKS=0;TICKS<4;TICKS++)                         
        {
         lcd.setCursor(0,0);
         lcd.print("-M.PAUSA------------");
         verDISPLAY();
         delay(350);
  
         if(digitalRead (START_button) == 0)
           {
            lcd.clear();
            if (BANDERA_PARADA_R1 == 1)
              {
               BANDERA_PARADA_R1 = 0;
               BANDERA_R1 == 1;
               Timer1.start();
               modo = 80;
               R_1();
              }
            }
         if(digitalRead (CLEAR_button) == 0){loop();}
        lcd.clear();
     for(TICKS=0;TICKS<4;TICKS++)                         
        {
         lcd.setCursor(0,0);
         lcd.print("-M.PAUSA------------");
         lcd.setCursor(0,1);
         lcd.print("TIEMPO DETENIDO     ");
         lcd.setCursor(0,2);
         lcd.print("paraSEGUIR-> (ENTER)");
         lcd.setCursor(0,3);
         lcd.print("--------------------");
         delay(350);
      
         if(digitalRead (ENTER_button) == 0)
           {
            lcd.clear();
            if (BANDERA_PARADA_R1 == 1)
              {
               BANDERA_PARADA_R1 = 0;
               Timer1.start();
               modo = 80;
               R_1();
              }
            }
          if(digitalRead (CLEAR_button) == 0) {loop();}   // salimos del programa y vamos al loop principal
        }
         if(modo==70)    implementamos la variable modo para usarla como bandera y volver al principio del            subprograma
        {
         PARADA();
        }
   }
```
Si bien, se puede hacer un simple contador y comparar los valores de dos variables, hay que tener en cuenta que esta es la base / un extracto de otro proyecto de mas envergadura. Como verán, puede parecernos un poco extenso y engorroso, pero si prestamos atención a los comentarios del sketch, no es para nada complicado replicarlo y sumarle mas relay al controlador.... Les dejo una vista ampliada de como quedaría:
Video Ejemplo de un Control Temporizado de 4 canales con función de salida en Cascada, Secuencial y Paso a Paso ( https://www.youtube.com/watch?v=KuOlEup2TWc )

[![](https://markdown-videos.deta.dev/youtube/KuOlEup2TWc)](https://www.youtube.com/watch?v=KuOlEup2TWc)

