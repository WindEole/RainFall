WALKTHROUGH LEVEL2 

---> ls -la
total 17
dr-xr-x---+ 1 level2 level2   80 Mar  6  2016 .
dr-x--x--x  1 root   root    340 Sep 23  2015 ..
-rw-r--r--  1 level2 level2  220 Apr  3  2012 .bash_logout
-rw-r--r--  1 level2 level2 3530 Sep 23  2015 .bashrc
-rwsr-s---+ 1 level3 users  5403 Mar  6  2016 level2
-rw-r--r--+ 1 level2 level2   65 Sep 23  2015 .pass
-rw-r--r--  1 level2 level2  675 Apr  3  2012 .profile

=> nous disposons d'un fichier level2, appartenant à level3. Pour les 
permissions user/group/other :
- permission s sur la partie user : bit SUID (setuid) activé donc il ne
	s'exécute qu'avec les privilèges du owner (ici level3) et non avec les
	privilèges du user (ici level2)
- permission s sur la partie group : bit GUID (getuid) activé sur 
	l'execution. Quand ce fichier est exécuté, le processus s'exécute avec
	les permissions du groupe owner (level3) en plus de celles de l'owner
	du fichier.
- permission + : des ACL (listes de contrôle d'accès) sont appliquées.

---> getfacl level2
#file: level2
#owner: level3
#group: users
#flags: ss-
user::rwx
user:level2:r-x
user:level3:r-x
group::---
mask::r-x
other::---