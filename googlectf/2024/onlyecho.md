# OnlyEcho

**CTF:** GoogleCTF 2024  
**Category:** misc  
**Original writeup:** <https://medium.com/@harryfyx/googlectf-quals-2024-only-echo-97f20db355cc> — by **harryfyx**. All credit goes to the original author; this is an archived copy.

---

We are presented with a script that executes commands with a check. It basically parses the user input in bash Abstract Syntax Tree (AST), and try to only allow echo as a command. AST is parsed by a javascript library called bash-parser.

const check = ast => {

if (typeof(ast) === 'string') {

return true;

}

for (var prop in ast) {

if (prop === 'type' && ast[prop] === 'Redirect') {

return false;

}

if (prop === 'type' && ast[prop] === 'Command') {

if (ast['name'] && ast['name']['text'] && ast['name']['text'] != 'echo') {

console.log('=== bad command ===');

return false;

}

}

if (!check(ast[prop])) {

return false;

}

}

return true;

};

Solution

We can achieve code execution through arithmetic expansion. This would pass the check since arithmetic expansion is not seen as “command”.

But it will not work for this challenge, due to bash-parser bug. Maybe this is because it does not think non-arithmetic syntax can exist there. This itself is a hint what this library is assuming, and what hackers want to see.

Let’s see the check again. Notice that for commands, it requires its ast[‘name’][‘text’] to be “echo”, if that field exists. We can try to find some senarios that such field does not exist.

Checking out bash-parser’s documentation, we see that name is not always there, and there is even an example.

Represents a builtin or external command to execute. It could optionally have a list of arguments, stream redirection operation and environment variable assignments. name properties is a Word that represents the name of the command to execute. It is optional because Command could represents bare assignment, e.g. VARNAME = 42; . In this case, the command node has no name.

We rewrite the code execution part with assignment syntax.

You can replace the url with your burp collaborator or other free tools.

Takeaways

Offense:

carefully read the documentation

learn more about weird syntax to bypass things

look for differences of processing, like how bash-parser understands a command, and how bash actually runs a command

carefully review the check logic, and realize what assumption is the author making
