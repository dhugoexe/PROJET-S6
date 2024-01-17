## I.	Partie software



1. #### Introduction



Dans le cadre de notre projet, la partie software a consisté à développer un programme capable de:

- Récupérer des données de position GPS reçue par un module GPS (*Seeed Studio 113020003*) via une communication UART.
- Traiter ces données et en extraire une position sous forme de coordonnées polaires (longitude, latitude et altitude).
- Afficher ces données sur un écran OLED, piloté par le bus I2C du microcontrôleur.
- En parallèle, gérer un module de transmission RF (le *SI4712*) dans l'objectif de transmettre des informations (notamment la position du système) au monde.



#### 2.	Récupérer et traiter des données de position GPS



Le module GPS, une fois alimenté va chercher à déterminer sa position en récupérant le signal de trois satellites. Avec ces trois signaux et avec un peu de triangulation, il est facile d'obtenir une position précise assez rapidement.

Le module GPS respecte la norme NMEA 0183. Ainsi, on sait déjà d'avance sous quelle forme le module va nous renvoyer les données de position. Dans le cadre de cette norme, on ne souhaite récupérer que les trames **GGA** (Global Positioning System Fix Data), de la forme suivante:

```
$GPGGA,064036.289,4836.5375,N,00740.9373,E,1,04,3.2,200.2,M,,,,0000*0E

$GPGGA       : Type de trame
064036.289  9: Trame envoyée à 06 h 40 min 36 s 289 (heure UTC)
4836.5375,N  : Latitude 48,608958° Nord = 48° 36' 32.25" Nord
00740.9373,E,: Longitude 7,682288° Est = 7° 40' 56.238" Est
1            : Type de positionnement (le 1 est un positionnement GPS)
04           : Nombre de satellites utilisés pour calculer les coordonnées
3.2          : Précision horizontale ou HDOP (Horizontal dilution of precision)
200.2,M      : Altitude 200,2, en mètres
,,,,,0000    : D'autres informations peuvent être inscrites dans ces champs
*            : séparateur de checksum
0E           : Somme de contrôle de parité, un simple XOR sur les caractères entre $ et *
```



Le programme cherche analyse toute les trames reçu dans le but de garder qu'une trame de ce type (car il y a également d'autres types de trames comme la trame **RMC**,  utilisés pour la navigation des bateaux). On souhaite doit également éliminer les trames vide ou incomplète: Le signal reçu est très faible, car il a été atténué par l'atmosphère et les nuages, il arrive donc que des données se perdent en chemin. Un check-sum est prévu dans la trame pour vérifier si elle est complète ou non. 

Notez que le module GPS rencontre de nombreux problèmes pour trouver sa position lors des journées très nuageuse. 



###### CODE 

```c
int main(void)
{

UART_HandleTypeDef huart4;

char gpsTrame[]= "$GPGGA,";

uint8_t gpsDataReceive[64] = {};
HAL_StatusTypeDef gpsData;

while (1)
{
  
  do{
		  gpsData = HAL_UART_Receive(&huart4, gpsDataReceive, 64, 200);

	  }
	  while(gpsData != HAL_OK);

	  if (strstr(gpsDataReceive, gpsTrame) != NULL)
	  {
		  if (strstr(gpsDataReceive, "N") != NULL)
		  {
			  if (strstr(gpsDataReceive, "E") != NULL)
			  {
				  char* N = strchr(gpsDataReceive, 'N');
				  int Npos = (int)(N - (char*)gpsDataReceive);

				  char* E = strchr(gpsDataReceive, 'E');
				  int Epos = (int)(E - (char*)gpsDataReceive);
          
				  char* M = strchr(gpsDataReceive, 'M');
				  int Mpos = (int)(M - (char*)gpsDataReceive);

				  if (Npos == 28 && Epos == 41 && Mpos == 57)
				  {
					  char* Latitude = extractSubstring(gpsDataReceive, Npos - 10, Npos);
					  char* Longitude = extractSubstring(gpsDataReceive, Epos - 11, Epos);
					  char* Altitude = extractSubstring(gpsDataReceive, Mpos - 5, Mpos);
				  }
			  }
		  }
}
}
  
char* extractSubstring(const char* inputString, int firstCharPos, int lastCharPos) 
{
    if (inputString == NULL || firstCharPos < 0 || lastCharPos < firstCharPos) {
        return NULL;
    }

    int length = lastCharPos - firstCharPos + 1;
    char* outputString = (char*)malloc((length + 1) * sizeof(char));

    if (outputString == NULL) {
        return NULL;
    }

    strncpy(outputString, inputString + firstCharPos, length);
    outputString[length] = '\0';

    return outputString;
}
```



Ce code simple nous permet de récupérer uniquement les trames complètes. Pour faire simple, on évalue la justesse de la trame en cherchant dans la trame reçue des éléments caractéristiques de cette dernière, comme la position dans la trame de certains caractères. Cette méthode peut sembler de prime abord un peu simpliste, voir incomplète mais dans la réalité des faits, le check-sum n'est jamais à vérifier, car toujours juste (pour une trame non vide). De plus le fait d'utiliser des caractères de repère nous permet d'extraire plus facilement les données de position. 



Le microcontrôleur communique avec le module via la liaison UART. Pour mémoire, la liaison UART est une liaison série asynchrone, communiquant sous forme de trame de données  (comme ci-dessous) avec un périphérique:



![img](https://upload.wikimedia.org/wikipedia/commons/thumb/0/0e/Constitution_trame_uart.png/600px-Constitution_trame_uart.png)

Pour veiller au bon fonctionnement du module, il est important de fixer la vitesse de communication à 9600 bauds (soit 9600 bits/s).



#### 2.	Afficher des données sur l'écran OLED 



Pour gagner du temps, nous avons utlisé pour l'affichage de données sur l'écran une bibliothèque de police d'écriture: l'utilisation d'une telle bibliothèque nous évite de devoir gérer l'affichage de données pixels par pixels. Ainsi nous avons des fonctions prêtes à l'emploi pour écrire du texte, tracer des formes géométriques simples, actualiser l'écran ou même effacer tout le contenu affiché. Pour plus de détails sur cette libraire, merci de vous référer au code complet disponible sur GitHub. 

Voici un exemple comment se passe l'affichage des données GPS sur l'écran: 

```c
ssd1306_Init();

ssd1306_Fill(Black);
ssd1306_SetCursor(0, 0);
ssd1306_WriteString("---GPS POSITION---", Font_6x8, White);

ssd1306_SetCursor(5, 15);
ssd1306_WriteString("Lat:", Font_6x8, White);
ssd1306_SetCursor(40, 15);
ssd1306_WriteString(Latitude, Font_6x8, White);
ssd1306_SetCursor(5, 30);
ssd1306_WriteString("Long:", Font_6x8, White);
ssd1306_SetCursor(40, 30);
ssd1306_WriteString(Longitude, Font_6x8, White);
ssd1306_SetCursor(5, 45);
ssd1306_WriteString("Alt:", Font_6x8, White);
ssd1306_SetCursor(40, 45);
ssd1306_WriteString(Altitude, Font_6x8, White);

ssd1306_UpdateScreen();
```

Information pratique sur la bibliothèque: librairie **ssd1306** pour **STM32** (https://github.com/afiskon/stm32-ssd1306).



L'écran est contrôlé via le bus I2C. Un peu plus bas se trouve une partie expliquant le fonctionnement du bus I2C un peu plus en détail. 



#### 3.	Communiquer avec le monde: utilisation du module radio 



L'objectif de cette partie a été d'élaborer un programme capable de contrôler l'émission radio ( à 107.5 MHz) d'un signal sonore. 

L'émetteur radio est fonctionne via le bus I2C. Un peu plus bas se trouve une partie expliquant le fonctionnement du bus I2C un peu plus en détail. Voici un algorigramme pour mieux comprendre comment utiliser d'un point de vue logiciel le module:

~~~flow
```flow
st=>start: Start
op=>operation: Reset
op2=>operation: POWER UP (command 0x01)
op3=>operation: Set FM Transmit Frequency (command 0x30)
op4=>operation: GET_INT_STATUS 
(command 0x14) 
op5=>operation: Call TX_TUNE_STATUS
with INTACK bit set
(command 0x33) 
op6=>operation: Set Transmit Power
(command 0x31) 
op7=>operation: GET_INT_STATUS 
(command 0x14) 
op8=>operation: Call TX_TUNE_STATUS
with INTACK bit set
(command 0x33) 
end=>operation: Transmitting

st->op->op2->op3->op4->op5->op6->op7->op8->end->

```
~~~



###### CODE

```c
#define RADIO_ADDRESS 198

I2C_HandleTypeDef hi2c1;

uint8_t power_up[3] = {0x01, 0x12, 0x50}; 
uint8_t tuneFreq[4] = {0x30, 0x00, 0x2A, 0x30};
uint8_t intStatus[1] = {0x14};
uint8_t tuneStatus[2] = {0x33, 0x01};
uint8_t tunePower[5] = {0x31, 0x00, 0x00, 115, 0x00};

uint8_t power_up_res[3] = {};
uint8_t hal_res[1] = {};
uint8_t tx_res[8] = {};

// RESET RADIO
  HAL_GPIO_WritePin(GPIOE,GPIO_PIN_15,0);		
  HAL_Delay(500);
  HAL_GPIO_WritePin(GPIOE,GPIO_PIN_15,0);		
  HAL_Delay(500);
  HAL_GPIO_WritePin(GPIOE,GPIO_PIN_15,1);
  HAL_Delay(50);


  // POWER UP RADIO
  dummy = HAL_I2C_Master_Transmit(&hi2c1, RADIO_ADDRESS, power_up, 3, -1);
  do{
	  dummy = HAL_I2C_Master_Receive(&hi2c1, RADIO_ADDRESS, power_up_res, 1, -1);
	  HAL_Delay(100);
  }while(power_up_res[0]!=0x80);


  // SET TX FREQ
  dummy = HAL_I2C_Master_Transmit(&hi2c1, RADIO_ADDRESS, tuneFreq, 4, -1);
  dummy = HAL_I2C_Master_Receive(&hi2c1, RADIO_ADDRESS, hal_res, 1, -1);
  HAL_Delay(1000);

  //GET INT STATUS
  dummy = HAL_I2C_Master_Transmit(&hi2c1, RADIO_ADDRESS, intStatus, 1, -1);
  do{
	  dummy = HAL_I2C_Master_Receive(&hi2c1, RADIO_ADDRESS, hal_res, 1, -1);
	  HAL_Delay(1000);
    }while(hal_res[0] !=0x81);

  HAL_Delay(1000);
  //TX STATUS
  dummy = HAL_I2C_Master_Transmit(&hi2c1, RADIO_ADDRESS, tuneStatus, 2, -1);
  dummy = HAL_I2C_Master_Receive(&hi2c1, RADIO_ADDRESS, tx_res, 8, -1);

  HAL_Delay(1000);
  //SET TX POWER
  dummy = HAL_I2C_Master_Transmit(&hi2c1, RADIO_ADDRESS, tunePower, 5, -1);
  dummy = HAL_I2C_Master_Receive(&hi2c1, RADIO_ADDRESS, hal_res, 1, -1);
  HAL_Delay(1000);

  //GET INT STATUS
  dummy = HAL_I2C_Master_Transmit(&hi2c1, RADIO_ADDRESS, intStatus, 1, -1);
    do{
  	  dummy = HAL_I2C_Master_Receive(&hi2c1, RADIO_ADDRESS, hal_res, 1, -1);
    	  HAL_Delay(1000);
      }while(hal_res[0]!=0x81);

//TX STATUS
dummy = HAL_I2C_Master_Transmit(&hi2c1, RADIO_ADDRESS, tuneStatus, 2, -1);
do{
		dummy = HAL_I2C_Master_Receive(&hi2c1, RADIO_ADDRESS, tx_res, 8, -1);
		HAL_Delay(100);
		}while(tx_res[0]!=0x80);
```



Cette partie du programme nous a donné beaucoup de fil à retordre: la documentation du composant était relativement complexe, et parfois incomplète, nous avons avancé en tâtonnant. C'est notamment le `GET INT STATUS 0x14` qui nous a posé problème, car il ne nous retournait jamais le bon code de retour, à savoir `0x81`. Nous avons également utilisé un analyseur de spectre pour savoir si le module émettait quelque chose, et de vérifier si ce n'était pas notre récepteur qui ne fonctionnait pas.

Ci-dessous se trouve une capture de l'analyseur de spectre lors d'une émission:



// Photo



On remarque bien un pique au niveau de 107.5 MHz.

**Remarque**: ce code actuel fonctionne uniquement en mode Debug, il doit avoir un problème de délai à respecter entre chaque instruction envoyée au module sur le bus I2C. 



#### 4.	Le bus I2C



Le bus I2C (Inter-Integrated Circuit) est un protocole de communication série conçue pour permettre la communication entre plusieurs dispositifs électroniques et un microcontrôleur, en utilisant seulement deux fils : une ligne de données (SDA) et une ligne d'horloge (SCL). 

Le bus I2C fonctionne selon un principe maître-esclave, où un seul  périphérique maître contrôle la communication avec les périphériques esclaves. Le périphérique maître initie toutes les transactions et  génère l'horloge pour synchroniser les transferts de données. Les périphériques esclaves,quant à eux, répondent aux demandes du maître.

Une trame I2C est composé d'un bit de start et d'un bit d'acquittement.

Le bus I2C est utilisé dans notre projet pour interagir avec des périphériques comme l'écran OLED ou l'émetteur radio. 





#### 5.	Amélioration possible 



Bien que quasi complètement fonctionnel, le programme peut toutefois être optimisé, voici quelques pistes possibles:

- Utilisation d'un seul bus I2C: Un unique bus permet la communication avec 128 périphériques. Dans notre cas on utilise un bus par périphérique (soit deux bus I2C).
- La fonction `main()` commence à être un peu lourde: il serait donc judicieux de faire d'autres fichiers c pour écrire des fonctions qui seront appelées dans la fonction `main()`.
- Pour la vérification des trames GPS, l'utilisation d'un masque peut-être judicieux est plus compacte.



L'objectif final était de coder les données GPS en morse pour les transmettre par radio. Cela n'est pas fondamentalement compliqué, mais demande du temps (que nous n'avions malheureusement pas). Pour réaliser cette tâche, il faut:

1. Définir une table de caractère alphanumérique et de caractère morse (l'ordre alphabétique est essentiel).

2. Formater la chaine de caractère à coder en majuscule. 

3. Pour chaque caractère de la chaine à coder, on associe à celui-ci un numéro (de place dans l'ordre alphabétique) pour pouvoir l'associer à un caractère morse, puis on stocke le caractère en morse dans une nouvelle chaine de caractères. 

4. Pour chaque caractère de la chaine codé, suivant si c'est un '.' ou un '-', on définit sur un pin en état haut pendant `dotDelay` ou `lineDelay`. On définie entre chaque état haut, un retour à l'état bas du pin pendant `lowDelay`.

   

