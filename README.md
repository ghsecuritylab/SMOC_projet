# SMOC Projet 2020

Github repository <https://github.com/2016floreza/SMOC_projet>

## Project description

The goal of our project is to create a player for a game of balls.

The game has a board, a referee and players and the objective is to push the others player out of the board without getting pushed out.

## Project composition

Our project is compose of two STM32F4_Discovery which realize two separate roles and a XBee car.

- The player master manage the player's game. It connects itself to a game, shows the game as given by the referee, and sends the acceleration of the player via the XBee.
- The accelerometer gets the acceleration of the card and sends it to the player master.
- The XBee which is a radio connection that allows the player to talk to the referee.

![alt text](https://i.imgur.com/zzuDaRo.jpg)

### Player master

The player master is the key component of a player. It is composed of:

- The card itself
- A connection to an XBee to communicate with the referee
- An ethernet connection to see on a screen how the game is going and your position
- A connection to the accelerometer
- A power supply

#### Connectors

Here are the connectors of our component:

![alt text](https://i.imgur.com/14SzJDT.jpg)

- Ethernet connection to a computer
- UART3 connection to connect with the XBee
- UART5 connection to connect with the accelerometer

![alt text](https://i.imgur.com/87Xh0GA.jpg)

#### Code

The player master code is in the player master repository. We used the encapsualtion code given to us during our project and we added the specifications to make them work with our project.

Because we did not have another card with an xbee module to execute the referee on, we decided to display the acceleration received from the accelerometer card on the web server as a proof of concept.

Here is the code that we added and why:

- an extern int8_t called acceleration which is used to pass the value received to the web server
- A buffer called received_cars that contains to chars : one to indicate whether the value is an "X" or "Y" acceleration, and one to indicate the value.

```c=
char print_flag = 0;
extern int8_t acceleration_x;
int8_t acceleration_x = 0;
extern int8_t acceleration_y;
int8_t acceleration_y = 0;
char received_cars[2];
char buffer[100];
```

- The ISR for UART receives an additional handler for UART5 which sets the print_flag to 1

```c=
if(huart->Instance == UART5){
  print_flag = 1;

    //on vérifie qu'on a bien reçu des message avec une  qui clignote
      if(HAL_UART_Receive_IT( &huart5, &received_cars, 2 ) == HAL_OK){
          HAL_GPIO_TogglePin( GPIOD, LD4_Pin);
      }

      //on relance l'écoute de l'UART
      HAL_UART_Receive_IT( &huart5, &buffer, 1 );
  }
```

- In the main while(1) loop, we check whether the flag is on. If it is then parse the values received.

```c=
//si on a reçu une accélération
  if(print_flag == 1){
    print_flag = 0;

    if(received_cars[0] == "X"){
        acceleration_x = (int8_t)received_cars[1];
    }
    if(received_cars[0] == "Y"){
        acceleration_y = (int8_t)received_cars[1];
    }
  }
```

- In the web server fs_open_custom() function we add the path /read_acceleration.html which adds the acceleration value as a char on the web page.

```c=
else if ( strcmp( name, "/read_acceleration.html" ) == 0 ){
     HAL_GPIO_TogglePin( GPIOD, LD5_Pin);
     file->data = buffer;

     len = sprintf( buffer, (char)acceleration_x );

   }
```

In order to test our code, we connect to the web server that runs on the player master, and reload the page repeatedly while moving the accelerometer. if the value changes, then we are indeed receiving accelerations from the accelerometer.

### Accelerometer

The accelerometer is here to guide our ball in the board. It is composed of:

- The card itself
- A connection to the player master

#### Connectors

Here are the connectors of our component:

![alt text](https://i.imgur.com/btSqDSW.png)

- UART3 connection to connect with the player master

![alt text](https://i.imgur.com/eCgvbAS.jpg)

#### Code

In the folder accelerometer, in the `/Src` folder, in the `main.c`, the while loop is sending continuously the acceleration to the player master.

```c=
  while (1)
  {
    /* USER CODE END WHILE */
    MX_USB_HOST_Process();

    /* USER CODE BEGIN 3 */
    if(BSP_ACCELERO_Init() != HAL_OK)
      {
        /* Initialization Error */
        Error_Handler();
      }

      while(1)
      {
        /* Accelerometer variables */
        int8_t buffer[3] = {0};
        int8_t xval, yval = 0x00;

        /* Read Acceleration */
        BSP_ACCELERO_GetXYZ(buffer);

        xval = buffer[0];
        yval = buffer[1];

        if((ABS(xval))>(ABS(yval)))
        {
            if(xval > ThresholdHigh)
            {
            HAL_UART_Transmit( &huart3, ["X", (char)100], 1, 1000 );
            HAL_Delay(10);
            }
            else if(xval < ThresholdLow)
            {
            HAL_UART_Transmit( &huart3, ["X", (char)-100], 1, 1000 );
            HAL_Delay(10);
            }
            else
            {
            HAL_UART_Transmit( &huart3, ["X", (char)xval], 1, 1000 );
            HAL_Delay(10);
            }
        }
        else
        {
            if(yval < ThresholdLow)
            {
            HAL_UART_Transmit( &huart3, ["Y", (char)100], 1, 1000 );
            HAL_Delay(10);
            }
            else if(yval > ThresholdHigh)
            {
            HAL_UART_Transmit( &huart3, ["Y", (char)-100], 1, 1000 );
            HAL_Delay(10);
            }
            else
            {
            HAL_UART_Transmit( &huart3, ["Y", (char)yval], 1, 1000 );
            HAL_Delay(10);
            }
        }
      }
  }
```

### XBee

The XBee is connected to the player master and allows it to communicated with the referee.

#### Connectors

Here are the connectors of the XBee:

![alt text](https://i.imgur.com/CRlMAvm.jpg)

## Difficulties

During our project, we encountered a lot of problems and managed to resolved them. They were all related to a compatibility problem between our software and hardware. For example, we didn't manage to make the xbee work until we updated the card.
