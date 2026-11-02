# Master_of_PHP

**CTF:** RealWorld 2019 Quals  
**Category:** misc  
**Original writeup:** <https://blog.bushwhackers.ru/pwning-master-of-php-like-its-real-real-world-ctf/> — by **blog.bushwhackers.ru**. All credit goes to the original author; this is an archived copy.

---

In our previous blog post, my teammate Emil has already published a solution for Master of PHP, however, I still want to share another way of solving this challenge, because I think it is quite interesting as well, and it doesn’t require usage of bug that was implanted into no_realworld_php. Instead, in this post we will use recently discovered curl 1-day to achieve stable code execution on the remote host.

Introduction

Okay, let’s do this one last time… The challenge implements a certain type of jail, where we are allowed to execute PHP code, but the execution of some functions is restricted by disable_functions , so you can’t get code execution straight ahead. Also, you can’t directly open any file on the system due to open_basedir being set to /tmp:/var/www/html . To get flag you have to get a stable code execution on the server and execute /readflag suid binary.

By looking into phpinfo() and Dockerfile we can see that our version of PHP was built with support of different libraries, but in particular, I was attracted by libcurl. Since I am subscribed to amazing oss-security mailing list, I noticed curl advisory about heap buffer overflow vulnerability in TFTP protocol published couple of days ago. As we can see from phpinfo(); our built-in version of curl supports TFTP protocol as well.

Review of CVE-2019-5482

Curl has pretty good security advisories, which provide us with useful information about the type of the bug, how to trigger it, when the bug was introduced and how it was patched. To debug this vulnerability I cloned curl source code from GitHub repo and did git reset --hard to some commit, so that version in phpinfo() and version in cloned repo would match. That way it was possible to compile curl with gdb symbols to LD_PRELOAD it later into apache process, so it would be so much easier to debug the process. After all, I realize that this was a terrible mistake and I describe it in more details in Mistakes were made chapter, as well as more correct way of the getting source code of curl to solve this challenge.

After going into the source code of TFTP protocol we can see that tftp_connect function allocates send buffer, receive buffer and tftp_state_data_t structure. From lib/tftp.c :

static CURLcode tftp_connect ( struct connectdata * conn , bool * done ) { tftp_state_data_t * state ; int blksize ; blksize = TFTP_BLKSIZE_DEFAULT ; state = conn -> proto . tftpc = calloc ( 1 , sizeof ( tftp_state_data_t )); if ( ! state ) return CURLE_OUT_OF_MEMORY ; if ( conn -> data -> set . tftp_blksize ) { blksize = ( int ) conn -> data -> set . tftp_blksize ; if ( blksize > TFTP_BLKSIZE_MAX || blksize < TFTP_BLKSIZE_MIN ) return CURLE_TFTP_ILLEGAL ; } if ( ! state -> rpacket . data ) { state -> rpacket . data = calloc ( 1 , blksize + 2 + 2 ); if ( ! state -> rpacket . data ) return CURLE_OUT_OF_MEMORY ; } if ( ! state -> spacket . data ) { state -> spacket . data = calloc ( 1 , blksize + 2 + 2 ); if ( ! state -> spacket . data ) return CURLE_OUT_OF_MEMORY ; } // . . .

From this piece of code we know that each buffer has default size, but if conn->data->set.tftp_blksize is set to non-zero and it passes some checks against nonsense values, then it becomes our blocksize. By grepping the code, we know that variable tftp_blksize is being set in lib/setopt.c . PHP curl API implements a rather limited version of real libcurl API, but this piece of code is within our reach. We can use curl_setopt PHP function to set blocksize for our TFTP connection before actually connecting to anything.

As advisory mentions, vulnerability is contained in the OACK processing. OACK stands for “option acknowledgement” as we can tell by taking a brief look into rfc2348. This functionality allows server to specify options it wants to use during the connection, and one of those options is blocksize. Client is required to agree upon options provided by server or reply with an error packet. After recieving each packet tftp_receive_packet routine parses first two bytes that indicate opcode of the corresponding handler. The following code of the function, which parses OACK packet, contains some changes to keep snippet small and readable:

static CURLcode tftp_parse_option_ack ( tftp_state_data_t * state , const char * ptr , int len ) { const char * tmp = ptr ; struct Curl_easy * data = state -> conn -> data ; state -> blksize = TFTP_BLKSIZE_DEFAULT ; while ( tmp < ptr + len ) { const char * option , * value ; tmp = tftp_option_get ( tmp , ptr + len - tmp , & option , & value ); if ( tmp == NULL ) { failf ( data , "Malformed ACK packet, rejecting" ); return CURLE_TFTP_ILLEGAL ; } if ( checkprefix ( option , TFTP_OPTION_BLKSIZE )) { long blksize ; blksize = strtol ( value , NULL , 10 ); // . . . // skipping out size checks of blksize // . . . state -> blksize = ( int ) blksize ; else if ( checkprefix ( option , TFTP_OPTION_TSIZE )) { // . . . } } return CURLE_OK ;

As you can see, in the first few lines state->blksize is being assign to default TFTP_BLKSIZE_DEFAULT value, which is equal to 512 bytes. Next, while loop parses all null terminated ascii key-value strings from inside the packet and checks if it matches certain string. But if key prefix doesn’t match any hardcoded value, then parser just ignores it. It is easy to see that if OACK packet doesn’t contain blocksize option, then state->blocksize would remain equal to TFTP_BLKSIZE_DEFAULT . Vulnerability becomes pretty obvious if you remember that state->blksize is used as length of the packet, which is received by recvfrom function in tftp_receive_packet , and the receiving buffer wasn’t reallocated at any point.

So, to trigger this vulnerability we would need to:

Create our own TFTP server. Create curl handle in PHP by using curl_init and set CURLOPT_TFTP_BLKSIZE to value less than TFTP_BLKSIZE_DEFAULT . Connect to our TFTP server from PHP code by using curl. Send OACK packet without blocksize option from server. Send next packet from server that would overflow receive buffer.

Exploitation

After happiness of the triggered bug subsides, it is time to realize all challenges of exploitation that we have here.

First of all, PHP and apache2 use different allocation algorithms, apart from curl, which uses default glibc malloc. So it wouldn’t be possible to change our glibc heap layout by allocating and freeing PHP objects.

There are not too many ways to manipulate heap during TFTP connection, because the only three allocations that are explicitly made here are the allocations of receive buffer, send buffer and state structure. Moreover, state structure is allocated before the receive buffer, so if allocations would arrange one after another, then the only thing that we can overflow is send buffer and something next to it.

If you attach debugger to the apache and try to reproduce the bug the same way as it was described in the previous chapter, you would see that during execution of tftp_connect there are no good places to fit our receive buffer heap chunk, since almost none of the chunks on the heap are free at this moment. As a result, allocations in ftp_connect would happen in the same order as they are in the code and the best thing that we can achieve is to overflow heap top chunk a.k.a wilderness. It opens up an opportunity for “house of force” exploitation, but this technique requires full control over size argument of malloc . This is not an option, because curl is rather good piece of software, and it is designed not to allow allocations of arbitrary sizes. Even it would allow such behavior, then it could be treated as a real “bug”, which probably corresponds to a “CWE-400: Uncontrolled Resource Consumption”.

The the last problem on the list is that we would probably require a memory leak during the process of exploitation. Considering that my amazing teammates have already shared with me a technique on how bypass open_basedir , we are able to read any file on the system, including /proc/self/maps , so open_basedir isn’t too much of a problem.

The main goal at this point is either to achieve arbitrary write by allocating state structure, right after our receive buffer, or we can overwrite some function pointer on the heap to achieve code execution. The second way works just fine.

Exploring code paths during connection termination would give us an answer to where do we find function pointers on the heap. In common scenario curl stores structures containing connection information in hash table. Each hash entry stores several pointers to a function that calculated the hash, a function to compare hashes and a pointer to destructor of a hash entry. Overwriting these hash entries may give us a pretty straight forward code execution, if we would manage to achieve good heap layout by using memory manipulation primitives correctly. Several hash entries are contained within struct Curl_multi structure, which is one of the main data structures to hold information about our connection. It is one of the first structures to be allocated after curl_exec has been called, so it will serve very well for our dark deeds.

The problem of achieving correct heap layout remains. In order to solve it, we can use other functions from PHP libcurl API that allow us to allocate and free objects on the same heap. Example of such function could be curl_setopt , which I end up using in the exploit.

First thing that you may notice in the code is an obvious allocation of buffer that can be allocated and reallocate of hold our buffer. Unfortunately, this primitive has really strong size limits. That means we won’t be able to manipulate smaller chunks, which is necessary, if we want to find our receive buffer a good place. Still, it can be used to either fill or create holes on the heap.

Another primitive that we would use is much harder to notice and requires diving deeper into the code. After browsing lots of option handlers I realized that CURLOPT_COOKIELIST is perfect for our goals. As mentioned above we would try to overwrite some function pointers, that are contained in struct Curl_multi . This structure has size of 0x1e0, so we would need to allocate receive buffer very close to it in order to overwrite the pointer.

Option CURLOPT_COOKIELIST requires cookie to be in a single line in Mozilla/Netscape or HTTP-style header and it will not accept malformed cookies. As we are able see from the code of the handler, several allocation are created there. First allocation malloc(strlen(arg)) is made (line 765) during call to strdup function, and it is free()-d at the end of the handler (line 782). There are several unimportant allocations made Curl_cookie_init function, but call to Curl_cookie_add (setopt.c:779) is very crucial for us. First things first, it allocates a struct Cookie by calling to calloc(), which has size of 0x60 bytes! If the supplied cookie is malformed, then function would free cookie structure and all its fields that were already parsed by this point. If none of the fields were successfully parsed, then it would just free our struct Cookie and return to the option handler, where is would free our strdup()-ed cookie string! If string supplied to this option would have length of 0x1e0, then we would be able to make two allocations of required sizes adjacent to each other and overflow the struct Curl_multi by allocating receive buffer of size 0x60. We will have to execute this primitive a couple of time, since most of our structures are allocated by using calloc , which doesn’t use tcache for some reason.

So this it. The last step we need is to leak /proc/self/maps and nail down our curl exploit in one script, since the second time our script gets executed, we may end up in a different apache worker, which would have a different memory layout. I implemented it by sending /proc/self/maps from PHP script by using TCP connection to my Python server, which was already handling TFTP connection on another port.

Mistakes were made

In this chapter I want to share the lesson I learned while solving this challenge.

As I have already pointed out, my attempt to clone curl from GitHub and try to match the version was a complete failure. Although, Ubuntu packages have a reputation of, let’s say, “unfresh”, it seems that maintainers are trying to care about the security of popular pieces of software. After critical vulnerability has been reported to curl, they share the patch with package maintainers. Even if in some distro an older version of libcurl is being maintained, security patch will be backported to it. On the day, when curl advisory goes public, maintainers publish security updated packages. Moreover, it seems that after security patch comes out, vulnerable binary packages are permanently deleted from the repository, so you don’t shoot yourself in a leg. Since docker image is based on Ubuntu 18.04 you can find Ubuntu Security Notice, which mentions the CVE, that we are trying to exploit. It also contains a link to a source package with patch description and a full version number: “7.58.0-2ubuntu3.8”. This version number consists of two parts. The first one: “7.58.0”, which is the version of curl. And second one: “2ubuntu3.8” the version of Ubuntu source package with backported fixes.

After I finished testing 90% ready-to-pwn exploit with local LD_PRELOAD-ed library with symbols, I finally decided to test it with real libcurl that was installed into my docker container, when it was built. And what do you know… It didn’t work… It is needless to describe my frustration, when I realized that Ubuntu package was updated the same day advisory was published in oss-security. Luckily, Emil has solved this challenge and saved the day.

After solving the challenge we ran dpkg -l | grep curl on the remote machine just for fun and guess what. The server was using unpatched version “ubuntu3.7”. This is an really epic fail, which resulted into a time loss of me and my teammate. I was able to read any file on the remote system, and I could have checked whether my exploit works with the remote library by LD_PRELOAD-ing it. After realizing that fact it was a matter of minutes to modify and finalize the exploit, but still…

This version mismatch probably happened, because challenge was already deployed on the server a week ago. So outdated container image is showing us a good level of preparations by the organizers.

Lesson learned. CTF RULE #37: Think less. Check everything you can at any point.

Conclusion

I would like to thank organizers for this amazing CTF, I’ve learned a lot from it. Real World CTF is definitely on my must-attend list.
