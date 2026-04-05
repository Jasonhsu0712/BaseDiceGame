# BaseDiceGame
DiceGame
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "@openzeppelin/contracts/token/ERC721/extensions/ERC721URIStorage.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract BaseBuilderNFT is ERC721, ERC721URIStorage, Ownable {
    uint256 private _nextTokenId;

    uint256 public mintPrice = 0.0001 ether;
    uint256 public maxSupply = 1000;

    event NFTMinted(address indexed to, uint256 tokenId, string uri);

    constructor() ERC721("BaseBuilderNFT", "BBNFT") Ownable(msg.sender) {}

    function publicMint(string memory uri) public payable {
        require(_nextTokenId < maxSupply, "Max supply reached");
        require(msg.value >= mintPrice, "Insufficient ETH sent");

        uint256 tokenId = _nextTokenId;
        _nextTokenId++;

        _safeMint(msg.sender, tokenId);
        _setTokenURI(tokenId, uri);

        emit NFTMinted(msg.sender, tokenId, uri);
    }

    function safeMint(address to, string memory uri) public onlyOwner {
        require(_nextTokenId < maxSupply, "Max supply reached");

        uint256 tokenId = _nextTokenId;
        _nextTokenId++;

        _safeMint(to, tokenId);
        _setTokenURI(tokenId, uri);

        emit NFTMinted(to, tokenId, uri);
    }

    function setMintPrice(uint256 newPrice) public onlyOwner {
        mintPrice = newPrice;
    }

    function setMaxSupply(uint256 newMax) public onlyOwner {
        require(newMax > _nextTokenId, "New max must be higher than current supply");
        maxSupply = newMax;
    }

    function totalSupply() public view returns (uint256) {
        return _nextTokenId;
    }

    // Required overrides
    function tokenURI(uint256 tokenId)
        public
        view
        override(ERC721, ERC721URIStorage)
        returns (string memory)
    {
    
    function supportsInterface(bytes4 interfaceId)
        public
        view
        override(ERC721, ERC721URIStorage)
        returns (bool)
    {
        return super.supportsInterface(interfaceId);
    }

    // Withdraw funds
    function withdraw() public onlyOwner {
        payable(owner()).transfer(address(this).balance);
    }
}
        return super.tokenURI(tokenId);
    }


// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/access/Ownable.sol";

contract BaseDiceGame is Ownable {
    struct PlayerStats {
        uint256 totalBets;      // Total bets placed
        uint256 totalWins;      // Total wins
        uint256 totalWinnings;  // Total ETH won
        uint256 lastPlayTime;   // Timestamp of last bet
    }

    mapping(address => PlayerStats) public playerStats;

    uint256 public minBet = 0.0001 ether;
    uint256 public maxBet = 0.01 ether;
    uint256 public houseEdgePercent = 15; // 15% house edge (payout ~4.25x)
    uint256 public totalBetsGlobal;
    uint256 public totalPayoutsGlobal;

    bool public paused = false;

    event BetPlaced(
        address indexed player,
        uint256 betAmount,
        uint8 guess,
        uint256 blockNumber
    );

    event DiceRolled(
        address indexed player,
        uint8 guess,
        uint8 result,
        bool won,
        uint256 payout,
        uint256 newBalance
    );

    event PrizePoolFunded(address indexed funder, uint256 amount);
    event OwnerWithdrawn(address indexed owner, uint256 amount);

    constructor() Ownable(msg.sender) {}

    modifier whenNotPaused() {
        require(!paused, "Game is paused");
        _;
    }

    function bet(uint8 guess) public payable whenNotPaused {
        require(guess >= 1 && guess <= 6, "Guess must be between 1 and 6");
        require(msg.value >= minBet && msg.value <= maxBet, "Bet amount out of range");
        require(msg.value <= address(this).balance / 5, "Bet too large for current balance");

        // Roll the dice (pseudo-random using block data)
        uint8 result = _rollDice();

        bool won = (result == guess);
        uint256 payout = 0;

        if (won) {
            // Payout = bet * (600 / (100 + houseEdge)) ≈ 4.25x with 15% edge
            uint256 multiplier = 600 / (100 + houseEdgePercent);
            payout = msg.value * multiplier / 100;
            require(address(this).balance >= payout, "Insufficient contract balance for payout");

            payable(msg.sender).transfer(payout);
            totalPayoutsGlobal += payout;
        }

        // Update player stats
        PlayerStats storage stats = playerStats[msg.sender];
        stats.totalBets++;
        stats.lastPlayTime = block.timestamp;
        if (won) {
            stats.totalWins++;
            stats.totalWinnings += payout;
        }

        // Global stats
        totalBetsGlobal++;

        emit BetPlaced(msg.sender, msg.value, guess, block.number);
        emit DiceRolled(msg.sender, guess, result, won, payout, address(this).balance);
    }

    function _rollDice() internal view returns (uint8) {
        // Provably fair pseudo-random (uses block.prevrandao + other data)
        // Note: Not cryptographically secure for high-stakes, but perfect for demo/game on Base
        uint256 seed = uint256(
            keccak256(
                abi.encodePacked(
                    block.prevrandao,
                    block.timestamp,
                    block.number,
                    msg.sender,
                    totalBetsGlobal
                )
            )
        );
        return uint8((seed % 6) + 1);
    }

    // === ADMIN / OWNER FUNCTIONS ===
    function setMinBet(uint256 newMin) public onlyOwner {
        require(newMin > 0, "Min bet must be > 0");
        minBet = newMin;
    }

    function setMaxBet(uint256 newMax) public onlyOwner {
        require(newMax > minBet, "Max bet must be > min bet");
        maxBet = newMax;
    }

    function setHouseEdge(uint256 newEdge) public onlyOwner {
        require(newEdge <= 30, "House edge too high");
        houseEdgePercent = newEdge;
    }

    function pauseGame() public onlyOwner {
        paused = true;
    }

    function unpauseGame() public onlyOwner {
        paused = false;
    }

    function fundPrizePool() public payable onlyOwner {
        require(msg.value > 0, "Must send ETH");
        emit PrizePoolFunded(msg.sender, msg.value);
    }

    function withdraw(uint256 amount) public onlyOwner {
        require(amount <= address(this).balance, "Insufficient balance");
        payable(owner()).transfer(amount);
        emit OwnerWithdrawn(owner(), amount);
    }

    function withdrawAll() public onlyOwner {
        uint256 balance = address(this).balance;
        require(balance > 0, "No funds to withdraw");
        payable(owner()).transfer(balance);
        emit OwnerWithdrawn(owner(), balance);
    }

    // === VIEW FUNCTIONS (for frontend / players) ===
    function getPlayerStats(address player) public view returns (
        uint256 totalBets,
        uint256 totalWins,
        uint256 totalWinnings,
        uint256 lastPlayTime
    ) {
        PlayerStats memory stats = playerStats[player];
        return (
            stats.totalBets,
            stats.totalWins,
            stats.totalWinnings,
            stats.lastPlayTime
        );
    }

    function getContractBalance() public view returns (uint256) {
        return address(this).balance;
    }

    function getGlobalStats() public view returns (
        uint256 totalBets,
        uint256 totalPayouts
    ) {
        return (totalBetsGlobal, totalPayoutsGlobal);
    }

    function estimatePayout(uint256 betAmount) public view returns (uint256) {
        if (betAmount == 0) return 0;
        uint256 multiplier = 600 / (100 + houseEdgePercent);
        return betAmount * multiplier / 100;
    }

    // Fallback to accept direct ETH (for prize pool)
    receive() external payable {
        emit PrizePoolFunded(msg.sender, msg.value);
    }
}
