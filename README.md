# Echoes-of-the-Forest
BaseEchoes
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * @title BaseEchoes - Echoes of the Forest (Pure Entertainment Edition)
 * @notice This is a highly creative, original on-chain "Echo Forest" game made exclusively for you.
 *         You shout a word into the forest, and the forest "echoes back" with a transformed, 
 *         poetic, or funny version of your word using on-chain randomness.
 *         Collect your echoes, build your Echo Journal, and unlock beautiful forest titles.
 *         Pure artistic entertainment — no gambling, no money required.
 */

contract BaseEchoes {
    address public owner;

    struct Echo {
        string originalWord;
        string echoedWord;
        string mood;           // Whisper, Laugh, Wisdom, Mystery, Joy
        uint256 timestamp;
    }

    // Player data
    mapping(address => uint256) public echoCount;
    mapping(address => Echo) public latestEcho;
    mapping(address => string) public playerTitle;
    mapping(address => uint256) public lastEchoTime;

    // Global stats
    uint256 public totalEchoes;

    // Beautiful forest titles
    string[7] public titles = [
        "Silent Wanderer",
        "Whisper Caller",
        "Forest Listener",
        "Echo Weaver",
        "Mystic Shouter",
        "Harmony Guardian",
        "Legendary Voice of the Woods"
    ];

    // Mood pool
    string[5] private moods = ["Whisper", "Laugh", "Wisdom", "Mystery", "Joy"];

    // Echo transformation styles
    string[8] private transformations = [
        " sings back ",
        " gently echoes ",
        " laughs and replies ",
        " whispers mysteriously ",
        " dances with ",
        " transforms into ",
        " poetically answers ",
        " magically becomes "
    ];

    event EchoReturned(
        address indexed player,
        string originalWord,
        string echoedWord,
        string mood,
        string title,
        uint256 echoNumber
    );

    constructor() {
        owner = msg.sender;
    }

    modifier onlyOwner() {
        require(msg.sender == owner, "Only owner can call");
        _;
    }

    /**
     * @notice Shout a word into the Echo Forest (free, 45 seconds cooldown)
     * @param _word Your word or short phrase (max 32 characters recommended)
     */
    function shoutIntoForest(string calldata _word) external {
        require(bytes(_word).length > 0 && bytes(_word).length <= 32, "Word too long or empty");
        require(
            block.timestamp >= lastEchoTime[msg.sender] + 45 seconds,
            "The forest needs time to rest (45s cooldown)"
        );

        lastEchoTime[msg.sender] = block.timestamp;
        echoCount[msg.sender] += 1;
        totalEchoes += 1;

        uint256 seed = uint256(
            keccak256(
                abi.encodePacked(
                    block.prevrandao,
                    block.timestamp,
                    msg.sender,
                    totalEchoes,
                    _word
                )
            )
        );

        // Generate echoed version
        string memory echoed = string(abi.encodePacked(_word, transformations[seed % 8], _word));

        // Add some magic for high seeds
        if (seed % 17 == 0) {
            echoed = string(abi.encodePacked("★ ", _word, " ★"));
        }

        string memory mood = moods[seed % 5];

        Echo memory newEcho = Echo({
            originalWord: _word,
            echoedWord: echoed,
            mood: mood,
            timestamp: block.timestamp
        });

        latestEcho[msg.sender] = newEcho;

        string memory newTitle = updateTitle(msg.sender);

        emit EchoReturned(
            msg.sender,
            _word,
            echoed,
            mood,
            newTitle,
            echoCount[msg.sender]
        );
    }

    function updateTitle(address player) internal returns (string memory) {
        uint256 count = echoCount[player];
        uint256 milestone = 0;

        if (count >= 100) milestone = 6;
        else if (count >= 70) milestone = 5;
        else if (count >= 45) milestone = 4;
        else if (count >= 25) milestone = 3;
        else if (count >= 10) milestone = 2;
        else if (count >= 3) milestone = 1;

        string memory newTitle = titles[milestone];

        if (keccak256(bytes(playerTitle[player])) != keccak256(bytes(newTitle))) {
            playerTitle[player] = newTitle;
        }

        return newTitle;
    }

    /**
     * @notice View your latest echo and stats
     */
    function getMyEcho() external view returns (
        string memory originalWord,
        string memory echoedWord,
        string memory mood,
        string memory title,
        uint256 totalEchoesByMe
    ) {
        Echo memory e = latestEcho[msg.sender];
        return (
            e.originalWord,
            e.echoedWord,
            e.mood,
            playerTitle[msg.sender],
            echoCount[msg.sender]
        );
    }

    /**
     * @notice View any player's echo stats
     */
    function getPlayerEcho(address player) external view returns (
        uint256 totalEchoesByPlayer,
        string memory title,
        string memory latestOriginal,
        string memory latestEchoed
    ) {
        Echo memory e = latestEcho[player];
        return (
            echoCount[player],
            playerTitle[player],
            e.originalWord,
            e.echoedWord
        );
    }

    /**
     * @notice Owner can reset player data (for testing)
     */
    function resetPlayer(address player) external onlyOwner {
        echoCount[player] = 0;
        playerTitle[player] = titles[0];
        lastEchoTime[player] = 0;
        delete latestEcho[player];
    }

    /**
     * @notice Withdraw any accidentally sent ETH
     */
    function withdraw() external onlyOwner {
        uint256 balance = address(this).balance;
        if (balance > 0) {
            payable(owner).transfer(balance);
        }
    }

    receive() external payable {}
}
