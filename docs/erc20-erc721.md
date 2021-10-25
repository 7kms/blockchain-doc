# ERC20, ERC721, ERC1155的区别和代码解释

在基于以太坊的智能合约开发过程中, 会涉及到一些合约的规范定义, 或者说标准协议. 其中最常用的有`ERC20`, `ERC721`, 和`ERC1155`. 本文主要解释其规范的定义, 以及[openzeppelin](https://docs.openzeppelin.com/contracts/4.x/tokens)对于该协议的实现. 

## ERC20

`erc20`是一种`同质化token`, `token`之间是完全等价的. `token`就是一个`uint256`类型的数字.

代币标准可以直接查看[定义](https://eips.ethereum.org/EIPS/eip-20), 这里列出[openzeppelin](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/v4.3.0/contracts/token/ERC20/ERC20.sol)的实现例子(只列出了常用方法, 全量文件可以查看github):

```ts
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract ERC20 is Context, IERC20, IERC20Metadata {
    mapping(address => uint256) private _balances;

    mapping(address => mapping(address => uint256)) private _allowances;

    uint256 private _totalSupply;

    string private _name;
    string private _symbol;

    /**
     * @dev See {IERC20-balanceOf}.
     */
    function balanceOf(address account) public view virtual override returns (uint256) {
        return _balances[account];
    }

    /**
     * @dev See {IERC20-transfer}.
     *
     * Requirements:
     *
     * - `recipient` cannot be the zero address.
     * - the caller must have a balance of at least `amount`.
     */
    function transfer(address recipient, uint256 amount) public virtual override returns (bool) {
        _transfer(_msgSender(), recipient, amount);
        return true;
    }

    /**
     * @dev Moves `amount` of tokens from `sender` to `recipient`.
     *
     * This internal function is equivalent to {transfer}, and can be used to
     * e.g. implement automatic token fees, slashing mechanisms, etc.
     *
     * Emits a {Transfer} event.
     *
     * Requirements:
     *
     * - `sender` cannot be the zero address.
     * - `recipient` cannot be the zero address.
     * - `sender` must have a balance of at least `amount`.
     */
    function _transfer(
        address sender,
        address recipient,
        uint256 amount
    ) internal virtual {
        require(sender != address(0), "ERC20: transfer from the zero address");
        require(recipient != address(0), "ERC20: transfer to the zero address");

        _beforeTokenTransfer(sender, recipient, amount);

        uint256 senderBalance = _balances[sender];
        require(senderBalance >= amount, "ERC20: transfer amount exceeds balance");
        unchecked {
            _balances[sender] = senderBalance - amount;
        }
        _balances[recipient] += amount;

        emit Transfer(sender, recipient, amount);

        _afterTokenTransfer(sender, recipient, amount);
    }
}

```
1. 属性分析: 这里重点查看其中的`_balances`.

```ts
mapping(address => uint256) private _balances;
```
`_balances`是一个`map`结构, 存储了`owner->token count`的映射关系. 所以很容易查到一个地址持有的代币数量

2. 方法分析: 可以查看实现`_transfer`方法

`_transfer`方法中, 除了校验代码之外(本文不分析校验, 只看逻辑), 核心代码:

```solidity
 uint256 senderBalance = _balances[sender];
 _balances[sender] = senderBalance - amount;
 _balances[recipient] += amount;
```
就是做了一个加减法, 所以`erc20token`称之为同质化代币, `token`之间是完全等价的. `token`就是一个`uint256`类型的数字.


## ERC721

`erc721`是一种`非同质化token(Non-Fungible Token)`, `token`之间是完全独立的, token是由链下传入的一个`uint256`类型的数字.

代币标准可以直接查看[定义](https://eips.ethereum.org/EIPS/eip-721), 这里列出[openzeppelin](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/v4.3.0/contracts/token/ERC721/ERC721.sol)的实现例子(只列出了常用方法, 全量文件可以查看github):

```ts
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;


/**
 * @dev Implementation of https://eips.ethereum.org/EIPS/eip-721[ERC721] Non-Fungible Token Standard, including
 * the Metadata extension, but not including the Enumerable extension, which is available separately as
 * {ERC721Enumerable}.
 */
contract ERC721 is Context, ERC165, IERC721, IERC721Metadata {
    using Address for address;
    using Strings for uint256;

    // Token name
    string private _name;

    // Token symbol
    string private _symbol;

    // Mapping from token ID to owner address
    mapping(uint256 => address) private _owners;

    // Mapping owner address to token count
    mapping(address => uint256) private _balances;

    // Mapping from token ID to approved address
    mapping(uint256 => address) private _tokenApprovals;

    // Mapping from owner to operator approvals
    mapping(address => mapping(address => bool)) private _operatorApprovals;
    
    /**
     * @dev See {IERC721-balanceOf}.
     */
    function balanceOf(address owner) public view virtual override returns (uint256) {
        require(owner != address(0), "ERC721: balance query for the zero address");
        return _balances[owner];
    }
    

    /**
     * @dev See {IERC721Metadata-tokenURI}.
     */
    function tokenURI(uint256 tokenId) public view virtual override returns (string memory) {
        require(_exists(tokenId), "ERC721Metadata: URI query for nonexistent token");

        string memory baseURI = _baseURI();
        return bytes(baseURI).length > 0 ? string(abi.encodePacked(baseURI, tokenId.toString())) : "";
    }

    /**
     * @dev Base URI for computing {tokenURI}. If set, the resulting URI for each
     * token will be the concatenation of the `baseURI` and the `tokenId`. Empty
     * by default, can be overriden in child contracts.
     */
    function _baseURI() internal view virtual returns (string memory) {
        return "";
    }


    /**
     * @dev Transfers `tokenId` from `from` to `to`.
     *  As opposed to {transferFrom}, this imposes no restrictions on msg.sender.
     *
     * Requirements:
     *
     * - `to` cannot be the zero address.
     * - `tokenId` token must be owned by `from`.
     *
     * Emits a {Transfer} event.
     */
    function _transfer(
        address from,
        address to,
        uint256 tokenId
    ) internal virtual {
        require(ERC721.ownerOf(tokenId) == from, "ERC721: transfer of token that is not own");
        require(to != address(0), "ERC721: transfer to the zero address");

        _beforeTokenTransfer(from, to, tokenId);

        // Clear approvals from the previous owner
        _approve(address(0), tokenId);

        _balances[from] -= 1;
        _balances[to] += 1;
        _owners[tokenId] = to;

        emit Transfer(from, to, tokenId);
    }

   
}

```
1. 属性分析: 这里重点查看其中的`_owners, _balances`.

  ```solidity
    // Mapping from token ID to owner address
    mapping(uint256 => address) private _owners;

    // Mapping owner address to token count
    mapping(address => uint256) private _balances;

  ```
   -  `_owners`: map结构, 存储的是`tokenId->owner`的对应关系
   -  `_balances`: map结构, 存储的是`owner->token account`的对应关系

   所以正常可以直接查出一个`erc721代币`所对应的`owner`, 以及`owner`所持有的代币个数. 但是不能直接查出一个`address(owner)`所拥有的所有代币的`tokenId`.

   这里的`token`实际上就是指代的`tokenID`是一个`uint256`类型. 它由外界传入, 也就是一个中心化服务器产生一个id, 存入去中心化的区块链上.

2. 方法分析: 
    - `_transfer`方法,其核心代码: 

      ```solidity
        _balances[from] -= 1; // 更新持有者的代表数量
        _balances[to] += 1;
        _owners[tokenId] = to; // 重新指定, 资产归属
      ```

    - `tokenURI`方法, 其核心代码: 

      ```solidity
        string memory baseURI = _baseURI(); // 其中_baseURI()是需要创建合约时自定义实现的, 用于链上与链下关联
        return bytes(baseURI).length > 0 ? string(abi.encodePacked(baseURI, tokenId.toString())) : "";
      ```

所以`erc721token`是一种`非同质化token(Non-Fungible Token)`, `token`是由链下传入的一个`uint256`类型的数字.

## ERC1155

`erc1155`严格来说, 它并不是`token`, 它是一种合约规范, 它所定义的接口的是为了更方便的管理多种代币. 

假设有如下场景: 为了项目需要, 现在要发10种`erc20token`和10种`erc721token`, 相当于要部署20个合约. 而且在调用这20个合约时, 是相互独立的, 不能同时对2种以上的代币进行转账等操作.

这时候`erc1155`规范(或者说协议), 是专门设计出来管理多个`erc20`,`erc721`合约的(可以实现同时查询多种token余额, 同时转账多种token). 接下来就从代码上来分析它的特别之处.

代币标准可以直接查看[定义](https://eips.ethereum.org/EIPS/eip-1155), 这里列出[openzeppelin](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/v4.3.0/contracts/token/ERC1155/ERC1155.sol)的实现例子(只列出了常用方法, 全量文件可以查看github):


```ts
/**
 * @dev Implementation of the basic standard multi-token.
 * See https://eips.ethereum.org/EIPS/eip-1155
 * Originally based on code by Enjin: https://github.com/enjin/erc-1155
 *
 * _Available since v3.1._
 */
contract ERC1155 is Context, ERC165, IERC1155, IERC1155MetadataURI {
    using Address for address;

    // Mapping from token ID to account balances
    mapping(uint256 => mapping(address => uint256)) private _balances;

    // Mapping from account to operator approvals
    mapping(address => mapping(address => bool)) private _operatorApprovals;

    // Used as the URI for all token types by relying on ID substitution, e.g. https://token-cdn-domain/{id}.json
    string private _uri;

    /**
     * @dev See {_setURI}.
     */
    constructor(string memory uri_) {
        _setURI(uri_);
    }


    /**
     * @dev See {IERC1155-balanceOf}.
     *
     * Requirements:
     *
     * - `account` cannot be the zero address.
     */
    function balanceOf(address account, uint256 id) public view virtual override returns (uint256) {
        require(account != address(0), "ERC1155: balance query for the zero address");
        return _balances[id][account];
    }

    /**
     * @dev See {IERC1155-balanceOfBatch}.
     *
     * Requirements:
     *
     * - `accounts` and `ids` must have the same length.
     */
    function balanceOfBatch(address[] memory accounts, uint256[] memory ids)
        public
        view
        virtual
        override
        returns (uint256[] memory)
    {
        require(accounts.length == ids.length, "ERC1155: accounts and ids length mismatch");

        uint256[] memory batchBalances = new uint256[](accounts.length);

        for (uint256 i = 0; i < accounts.length; ++i) {
            batchBalances[i] = balanceOf(accounts[i], ids[i]);
        }

        return batchBalances;
    }

   /**
     * @dev Transfers `amount` tokens of token type `id` from `from` to `to`.
     *
     * Emits a {TransferSingle} event.
     *
     * Requirements:
     *
     * - `to` cannot be the zero address.
     * - `from` must have a balance of tokens of type `id` of at least `amount`.
     * - If `to` refers to a smart contract, it must implement {IERC1155Receiver-onERC1155Received} and return the
     * acceptance magic value.
     */
    function _safeTransferFrom(
        address from,
        address to,
        uint256 id,
        uint256 amount,
        bytes memory data
    ) internal virtual {
        require(to != address(0), "ERC1155: transfer to the zero address");

        address operator = _msgSender();

        _beforeTokenTransfer(operator, from, to, _asSingletonArray(id), _asSingletonArray(amount), data);

        uint256 fromBalance = _balances[id][from];
        require(fromBalance >= amount, "ERC1155: insufficient balance for transfer");
        unchecked {
            _balances[id][from] = fromBalance - amount;
        }
        _balances[id][to] += amount;

        emit TransferSingle(operator, from, to, id, amount);

        _doSafeTransferAcceptanceCheck(operator, from, to, id, amount, data);
    }

    /**
     * @dev xref:ROOT:erc1155.adoc#batch-operations[Batched] version of {_safeTransferFrom}.
     *
     * Emits a {TransferBatch} event.
     *
     * Requirements:
     *
     * - If `to` refers to a smart contract, it must implement {IERC1155Receiver-onERC1155BatchReceived} and return the
     * acceptance magic value.
     */
    function _safeBatchTransferFrom(
        address from,
        address to,
        uint256[] memory ids,
        uint256[] memory amounts,
        bytes memory data
    ) internal virtual {
        require(ids.length == amounts.length, "ERC1155: ids and amounts length mismatch");
        require(to != address(0), "ERC1155: transfer to the zero address");

        address operator = _msgSender();

        _beforeTokenTransfer(operator, from, to, ids, amounts, data);

        for (uint256 i = 0; i < ids.length; ++i) {
            uint256 id = ids[i];
            uint256 amount = amounts[i];

            uint256 fromBalance = _balances[id][from];
            require(fromBalance >= amount, "ERC1155: insufficient balance for transfer");
            unchecked {
                _balances[id][from] = fromBalance - amount;
            }
            _balances[id][to] += amount;
        }

        emit TransferBatch(operator, from, to, ids, amounts);

        _doSafeBatchTransferAcceptanceCheck(operator, from, to, ids, amounts, data);
    }
```
1. 属性分析`_balances`

    ```solidity
        mapping(uint256 => mapping(address => uint256)) private _balances;
    ```

    从源码上来看, `_balances`是一个`嵌套map`结构, 相当于:

      ```js
      _balances = {
        "erc20/erc721 合约编号1": {
          "以太坊地址" : "token数量"
        },
         "erc20/erc721 合约编号2": {
          "以太坊地址" : "token数量"
        },
         "erc20/erc721 合约编号3": {
          "以太坊地址" : "token数量"
        },
        ...
      }
      ```
2. 方法分析

    - `balanceOf`和`balanceOfBatch`: 分别代表查询`单个token`的余额和查询`多个token`的余额
      核心代码:

      ```solidity
       function balanceOf(address account, uint256 id) public view virtual override returns (uint256) {
          require(account != address(0), "ERC1155: balance query for the zero address");
          return _balances[id][account]; // 获取嵌套map中的属性
        }
      function balanceOfBatch(address[] memory accounts, uint256[] memory ids)
        public
        view
        virtual
        override
        returns (uint256[] memory)
        {
            require(accounts.length == ids.length, "ERC1155: accounts and ids length mismatch");

            uint256[] memory batchBalances = new uint256[](accounts.length);
            // 循环调用balanceOf
            for (uint256 i = 0; i < accounts.length; ++i) {
                batchBalances[i] = balanceOf(accounts[i], ids[i]);
            }

            return batchBalances;
        }

      ```

      - `safeTransferFrom`和`safeBatchTransferFrom`: 内部逻辑与`balance`一致, 都是解析参数, 然后进行批处理.

所以`erc1155`规范是为了解决多个token的管理问题(理论上如果只发布1-2个token, 不用`erc1155`也没什么影响).

## 三者的配合使用

正常情况下, 在项目中需要同时发布`erc20`和`erc721token`的时候, 都会实现`erc1155`协议. 有关三者的配合使用, 将单独出文.


## 总结

本节主要列举了`erc20`,`erc721`,`erc1155`三种不同协议, 以及`openzeppelin`上面对它们的实现. `erc1155`是对前2种token的管理, 拥有单一合约控制多个token的能力.