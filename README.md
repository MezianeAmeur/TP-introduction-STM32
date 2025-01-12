# 2425_TP_FPGA-LINUX -Aliou LY & Bayazide BELKHIR

## TP 1 : Tutoriel Quartus

### Création d’un projet

### Création d’un fichier VHDL

### Fichier de contrainte
Un fichier de contrainte est nécessaire pour spécifier les connexions des broches du FPGA avec les composants externes (comme les LED, boutons, etc.).
### Programmation de la carte
### Modification du VHDL
### Faire clignoter une LED
L’horloge nommée `FPGA_CLK1_50` est conectée sur la broche `PIN_V11` .
le code suivant est equivalent a ce schema vhdl 
```vhdl
library ieee;
use ieee.std_logic_1164.all;

entity led_blink is
    port (
        i_clk   : in std_logic;  -- Entrée d'horloge
        i_rst_n : in std_logic;  -- Réinitialisation active bas
        o_led   : out std_logic  -- Sortie LED
    );
end entity led_blink;

architecture rtl of led_blink is
    signal r_led : std_logic := '0';  -- Signal interne pour l'état de la LED
begin
    process(i_clk, i_rst_n)
    begin
        if (i_rst_n = '0') then         -- Réinitialisation
            r_led <= '0';
        elsif (rising_edge(i_clk)) then -- Front montant de l'horloge
            r_led <= not r_led;        -- Inverse l'état de la LED
        end if;
    end process;

    o_led <= r_led;  -- Connecte le signal interne à la sortie
end architecture rtl;

```

![Schema LED](/TP1/sche.png)
![Schema LED](/TP1/schemaq.png)

Avce une frequence de `50 MHz` le clignottement est invisible . C'est pour cela q'on utilise un de pour diviser la frequence . Le compteur compte jusqu'a `5000000` pour avoir une frequence de 10Hz .
``` vhdl
begin
    process(i_clk, i_rst_n)
    variable counter : natural range 0 to 5000000 := 0;
    begin
        if (i_rst_n = '0') then         -- Réinitialisation
            r_led <= '0';
            counter := 0;
        elsif (rising_edge(i_clk)) then -- Front montant de l'horloge
            if(counter=5000000) then
                r_led <= '1';        -- Inverse l'état de la LED
                counter := 0;
            else
                r_led <= '0'; 
                counter := counter + 1 ;
            end if ;
        end if;
    end process;

    o_led <= r_led;  -- Connecte le signal interne à la sortie
end architecture rtl;
```
![schema 2 led ](/TP1/sch.png)
![schema 2 led ](/TP1/schmaq.png)
### 1.7 Chenillard !
Le chenillard consiste à faire défiler des LED de manière cyclique. Cela peut être réalisé en contrôlant une série de LEDs avec un compteur. Pour cela :
- Définissez un tableau de LEDs et un compteur pour gérer les indices.
- Faites défiler les LEDs d'une manière régulière en allumant successivement chaque LED du tableau.
- Testez le chenillard sur la carte et ajustez les paramètres si nécessaire.

## TP 2 : Petit projet - Bouncing ENSEA Logo

### **Analyse de l'entity du fichier `hdmi_generator.vhd`**

#### **Paramètres génériques**
Les paramètres définis dans la section `generic` permettent de configurer la résolution et les timings HDMI :

1. **Résolution** :
   - `h_res` : Largeur de l'image visible, exprimée en nombre de pixels (par défaut : 720).
   - `v_res` : Hauteur de l'image visible, exprimée en nombre de pixels (par défaut : 480).

2. **Timings HDMI** :
   - **Horizontal** :
     - `h_sync` : Durée du signal de synchronisation horizontal (en cycles d'horloge).
     - `h_fp` : Front Porch horizontal, temps entre la fin de l'image et le signal de synchronisation.
     - `h_bp` : Back Porch horizontal, temps entre la fin du signal de synchronisation et le début de l'image visible.
   - **Vertical** :
     - `v_sync` : Durée du signal de synchronisation vertical (en lignes).
     - `v_fp` : Front Porch vertical, temps entre la fin de l'image et le signal de synchronisation.
     - `v_bp` : Back Porch vertical, temps entre la fin du signal de synchronisation et le début de l'image visible.

#### **Unité des paramètres**
- `h_res`, `v_res`, `h_sync`, `h_fp`, `h_bp`, `v_sync`, `v_fp`, `v_bp` sont exprimés en cycles d'horloge ou en lignes selon leur contexte.

#### **Rôle de certains signaux**

1. **Sorties principales** :
   - `o_new_frame` : Passe à l'état haut pendant un cycle d'horloge à la fin de chaque trame.
   - `o_pixel_pos_x` et `o_pixel_pos_y` : Indiquent la position courante (X, Y) du pixel sur la zone d'affichage active.
   - `o_pixel_address` : Adresse unique d'un pixel, calculée avec la formule :
     ```
     o_pixel_address = o_pixel_pos_x + (h_res × o_pixel_pos_y)
     ```

2. **Signaux HDMI** :
   - `o_hdmi_hs` : Signal de synchronisation horizontal.
   - `o_hdmi_vs` : Signal de synchronisation vertical.
   - `o_hdmi_de` : Signal **Data Enable**, actif uniquement pour les pixels visibles.

3. **Signaux supplémentaires** :
   - `o_pixel_en` : Active un pixel pour l'affichage.
   - `o_x_counter` et `o_y_counter` : Compteurs pour les positions horizontales et verticales (dans les zones visibles).

### **Resulatat de la simulation**
En limitant les valeurs des paramètres de la résolution, nous avons simulé le résultat avec les paramètres suivants :
```vhdl
   
 -- paramètres du test (réduits pour la simulation)
 constant h_res      : natural := 5;  
 constant h_sync     : natural := 1;  
 constant h_fp       : natural := 1;  
 constant h_bp       : natural := 1;  
 
 constant v_res      : natural := 5;  
 constant v_sync     : natural := 1;  
 constant v_fp       : natural := 1;  
 constant v_bp       : natural := 1; 
```
Avec ces paramètres, nous obtenons le résultat attendu lors de la simulation. Cependant, l'implémentation ne fonctionne pas correctement en pratique parceque j'avais oublié de mettre cette ligne ```o_pixel_address <= r_pixel_counter``` de plus, le signal `r_pixel_address` ne devait pas être réinitialisé à zéro après chaque ligne.
![Logo ENSEA](/Simulation/sim.png)
Avec  le fichier fourni par le professeur.
![Logo ENSEA](/Simulation/simprof.png)
### **Implémentation sur le FPGA**
La mémoire utilisée a une taille de 95×95 et contient une image du logo de l'ENSEA en niveaux de gris. Elle est constituée de deux ports, mais ici, seul le port A est utilisé. L'adresse de la mémoire est calculée par la formule suivante : ` add <= x_count + 95*y_count ;`

```vhdl
    mem : entity work.dpram
        port map (
            i_clk_a => vpg_pclk,
            i_clk_b => vpg_pclk,
            i_addr_a => add,
            i_addr_b => add,
            o_q_a => data,
            o_q_b => open  
        );

```

Pour afficher l'image dans le coin supérieur gauche de l'écran, nous devons placer les données de la mémoire dans les pixels correspondant aux coordonnées allant de 0 à 95 pour les axes x et y, puis afficher 0 (noir) pour le reste de l'écran.

```vhdl
    begin
        if (x_count < 95 and y_count < 95) then
            HDMI_TX_D(23 downto 16) <= data;
        else
            HDMI_TX_D(23 downto 16) <= x"00";
        end if;
    end process;
```

Cette partie du code permet d'afficher l'image du logo dans la zone spécifiée tout en maintenant le reste de l'écran noir.
![Logo ENSEA](/TP2/tp2_fpga/Screenshot_20241216_171441.png)

---
