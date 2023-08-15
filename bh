// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "@openzeppelin/contracts/utils/Counters.sol";
import "@openzeppelin/contracts/utils/Strings.sol";
import "@openzeppelin/contracts/utils/Address.sol";
import "@openzeppelin/contracts/utils/cryptography/MerkleProof.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

// @author: twitter.com/alpcheff && twitter.com/devDwarf

contract BigintHandle is ERC721, ReentrancyGuard {

    using Counters for Counters.Counter;
    Counters.Counter private _tokenIds;
    mapping(string => bool) private _nameExists;
    mapping(uint256 => string) private _tokenNames;
    mapping(address => uint256) private primaryName;
    mapping(address => bool) public whitelistClaimed;
    mapping(uint256 => uint256) private _refferalAmounts;
    mapping(address => bool) public allowedContractAddress;

    string private _internalBaseURI;

    uint256 private _totalSupply;

    address private _owner;

    bytes32 public merkleRoot;

    bool public isFreeMintOpen;
    bool public isMintOpen;
    bool public isReferrerWithdrawOpen;

    event MintEvent(
        address indexed _walletAddress,
        string _name,
        uint256 _ref,
        uint256 _tokenId
    );

    modifier onlyOwner() {
        require(msg.sender == _owner, "Only the contract owner can call this function");
        _;
    }

    modifier onlyTokenOwner(uint256 tokenId) {
        require(ownerOf(tokenId) == msg.sender, "You are not the owner of this name");
        _;
    }

    modifier mintOpen() {
        require(isMintOpen, "Mint closed by Owner");
        _;
    }

    modifier freeMintOpen() {
        require(isFreeMintOpen, "FreeMint closed by Owner");
        _;
    }

    modifier referrerWithdrawOpen() {
        require(isReferrerWithdrawOpen, "ReferrerWithdraw closed by Owner");
        _;
    }

    modifier nameNonExist(string memory name) {
        require(!_nameExists[name], "Name already exists");
        _;
    }

    modifier onlyWallets() {
        require((!Address.isContract(msg.sender) && msg.sender == tx.origin) || allowedContractAddress[msg.sender], "Reverting, Method can only be called directly by user");
        _;
    }

    constructor() ERC721("BigintHandle", "BH") {
        _owner = msg.sender;
        isFreeMintOpen = false;
        isMintOpen = false;
        isReferrerWithdrawOpen = false;
    }

    function mintNFT(string memory name, uint256 refId) public payable mintOpen nameNonExist(name) nonReentrant onlyWallets {
        require(msg.sender == tx.origin, "Reverting, Method can only be called directly by user.");
        uint256 mintPrice = calculatePrice(name);
        require(refId >= 0 || refId < _totalSupply, "Invalid reference name");
        require(msg.value >= calculatePrice(name), "Insufficient payment");

        _tokenIds.increment();
        uint256 newNFTId = _tokenIds.current();
        _mint(msg.sender, newNFTId);
        _setTokenName(newNFTId, name);
        _nameExists[name] = true;
        _refferalAmounts[refId] =
            _refferalAmounts[refId] +
            (mintPrice * 15) /
            100;
        
        _totalSupply++;

        if (primaryName[msg.sender] < 1) {
            primaryName[msg.sender] = newNFTId;
        }
        emit MintEvent(msg.sender, name, refId, newNFTId);
    }

    function freeMintNFT(string memory name, bytes32[] calldata _merkleProof) public freeMintOpen nameNonExist(name) nonReentrant onlyWallets {
        require(!whitelistClaimed[msg.sender], "Address already claimed");
        bytes32 leaf = keccak256(abi.encodePacked(msg.sender));
        require(MerkleProof.verify(_merkleProof, merkleRoot, leaf), "Invalid proof");

        _tokenIds.increment();
        uint256 newNFTId = _tokenIds.current();
        _mint(msg.sender, newNFTId);
        _setTokenName(newNFTId, name);
        _nameExists[name] = true;

        _totalSupply++;

        whitelistClaimed[msg.sender] = true;

        if (primaryName[msg.sender] < 1) {
            primaryName[msg.sender] = newNFTId;
        }
        emit MintEvent(msg.sender, name, 0, newNFTId);
    }

    function referrerWithdraw(uint256 tokenId) public onlyTokenOwner(tokenId) referrerWithdrawOpen nonReentrant onlyWallets {
        require(_refferalAmounts[tokenId] > 0, "Referral commission is not enough");
        require(_refferalAmounts[tokenId] < address(this).balance, "Insufficient contract balance");
        (bool success, ) = payable(msg.sender).call{value: _refferalAmounts[tokenId]}("");
        require(success, "Transfer failed");
        _refferalAmounts[tokenId] = 0;
    }

    function calculatePrice(string memory name) public pure returns (uint256) {
        uint256 length = bytes(name).length;

        require(length >= 1 && length <= 28, "Invalid name length");

        if (length <= 3) {
            return 0.006 ether;
        } else if (length <= 6) {
            return 0.005 ether;
        } else {
            return 0.004 ether;
        }
    }

    function transferFrom(
        address from,
        address to,
        uint256 tokenId
    ) public virtual override {
        super.transferFrom(from, to, tokenId);
        if (primaryName[from] == tokenId) {
            _removePrimaryName(from);
        }
    }

    function safeTransferFrom(
        address from,
        address to,
        uint256 tokenId,
        bytes memory data
    ) public virtual override {
        super.safeTransferFrom(from, to, tokenId, data);
        if (primaryName[from] == tokenId) {
            _removePrimaryName(from);
        }
    }

    function _removePrimaryName(address walletAddress) internal {
        require(primaryName[walletAddress] != 0, "Primary name does not exist for the wallet address");
        delete primaryName[walletAddress];
    }

    function updatePrimaryNameId(uint256 tokenId) public onlyTokenOwner(tokenId) {
        primaryName[msg.sender] = tokenId;
    }

    function getPrimaryNameIdFromWallet(address walletAddress) public view returns (uint256) {
        return primaryName[walletAddress];
    }

    function totalSupply() public view returns (uint256) {
        return _totalSupply;
    }

    function checkNameExists(string memory name) public view returns (bool) {
        return _nameExists[name];
    }

    function getNameFromId(uint256 tokenId) public view returns (string memory) {
        return _tokenNames[tokenId];
    }

    function getReferralAmount(uint256 tokenId) public view returns (uint256) {
        return _refferalAmounts[tokenId];
    }

    function _baseURI() internal view virtual override returns (string memory) {
        return _internalBaseURI;
    }

    function tokenURI(uint256 tokenId) public view virtual override returns (string memory) {
        require(_exists(tokenId), "ERC721Metadata: URI query for nonexistent token");
        return
            bytes(_baseURI()).length > 0
                ? string(abi.encodePacked(_baseURI(), Strings.toString(tokenId)))
                : "";
    }

    function _setTokenName(uint256 tokenId, string memory name) internal virtual {
        require(_exists(tokenId), "ERC721Metadata: NAME set of nonexistent token");
        _tokenNames[tokenId] = name;
    }

    function ownerWithdraw(uint256 amount) public onlyOwner {
        require(amount <= address(this).balance, "Insufficient contract balance");

        (bool success, ) = payable(_owner).call{value: amount}("");
        require(success, "Transfer failed");
    }

    function setTokenBaseURI(string memory baseURI) public onlyOwner {
        _internalBaseURI = baseURI;
    }

    function updateOwner(address owner) public onlyOwner {
        _owner = owner;
    }

    function setMerkleRoot(bytes32 _newMerkleRoot) public onlyOwner {
        merkleRoot = _newMerkleRoot;
    }

    function toggleFreeMint() public onlyOwner {
        if (isFreeMintOpen) isFreeMintOpen = false;
        else isFreeMintOpen = true;
    }

    function toggleMint() public onlyOwner {
        if (isMintOpen) isMintOpen = false;
        else isMintOpen = true;
    }

    function toggleReferrerWithdraw() public onlyOwner {
        if (isReferrerWithdrawOpen) isReferrerWithdrawOpen = false;
        else isReferrerWithdrawOpen = true;
    }

    function addAllowedContractAddress(address contractAddress) public onlyOwner {
        allowedContractAddress[contractAddress] = true;
    }

    function removeAllowedContractAddress(address contractAddress) public onlyOwner {
        allowedContractAddress[contractAddress] = false;
        delete allowedContractAddress[contractAddress];
    }

    function getContractAmount() public view returns (uint256) {
        return address(this).balance;
    }
}
