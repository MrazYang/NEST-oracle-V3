## NEST oracle price checking

### Description
> The oracle price data is stored in the pricing contract. The pricing contract provides relating interfaces and implemented the charging logic. The address of pricing contract should be queried from voting contract.

### Note
1. `receive` method must be implemented in the contract for receiving the returned ETH.

```
receive() external payable
```

2. when calling pricing contract, the address must be obtained via NEST vote contract, in case of ineffectiveness after the address of pricing contract changes.

```
Nest_3_OfferPrice _offerPrice = Nest_3_OfferPrice(address(_voteFactory.checkAddress("nest.v3.offerPrice")));
```
3. The charging standard for the creation of each ERC20 oracle is the same, but may be modified by voting contract. When calling prices, charging standard may changes, to make sure the success callings , there are two methods.

> Method1：When calling, pay a larger amount of ETH first, the pricing contract will charge the actual fee according to the rules, the excess ETH will be returned to the calling contract, and the receiving of returned ETH should be handled by the calling contract. The specific amount of ETH paid first can be determined according to the upper limit that you can accept. 

> Method2：Before calling the price, check the corresponding  charging standard of ERC20 from the price contract, calculate the fee based on the charging rules, and then call the price.

```
    //  查看获取价格eth最少费用 
    function checkPriceCostLeast(address tokenAddress) public view returns(uint256) {
        return _tokenInfo[tokenAddress].priceCostLeast;
    }
    
    //  查看获取价格eth最多费用 
    function checkPriceCostMost(address tokenAddress) public view returns(uint256) {
        return _tokenInfo[tokenAddress].priceCostMost;
    }
    
    //  查看价格eth单条数据费用
    function checkPriceCostSingle(address tokenAddress) public view returns(uint256) {
        return _tokenInfo[tokenAddress].priceCostSingle;
    }
```
==4. Do not implement methods that can publicly check prices (to prevent proxy prices) within the contract. Any method of calling Nest oracle price data is only for internal use of the contract. If it includes a method of publicly checking the price, the contract will be voted by the community to be added to the price calling blacklist, and the contract will not be able to call the Nest oracle price. ==

5. Make sure the erc20 oracle is created and operated normally.

### Price generation instruction
#### Pricing block, validation block, effective price block

Pricing block number is 1，validation block number is between 2 and 26 (taker order available). When the number exceed 26, the pricing assets can be retrieved and the effective price block is 26. After the 26th block, the effective price in 26th will always be returned.
#### Price calculation with multiple offers in a block
The first offer：10ETH， 2000USDT 

The second offer：100ETH，3000USDT

The calculation of effective price (weighted average)：(10ETH + 100ETH) / (2000USDT + 3000USDT)

### Charging instruction
There are 2 types of price calling

1. Calling the latest price（return the latest price data）。
2. Calling history price（Backward from the latest price, can choose the number of prices and return price data array).

Charging
1. Calling the latest price,  payment of 0.001ETH(default).
2. Calling history price, a single data is 0.0001ETH(default)；payment of 10 prices for number less than 10；payment of 100 prices for number more than 100.

Distribution

20% of the payment is distributed to the last offering miner of the corresponding block number of the effective price, 80% of the payment is distributed to the bonus pool of corresponding Token.

### Method

#### Activation

The authorization should be activated before calling prices

##### Activation Instruction
There are two preconditions for using Nest oracle prices
1. 10000 Nest destructed in activation
2. wait for 1 day

Nest is required before activation in the third-party contract, but it does not necessarily transfer first, activation can be combined into one transaction。


```
function activation() public
```


#### Calling a single price
input parameter | description 
---|---
tokenAddress | the erc20 address for checking 

output parameter | description 
---|---
ethAmount | eth amount（total eth amount of offering） 
erc20Amount | erc20 amount（total erc20 amount of offering） 
blockNum | effective block number 

```
function updateAndCheckPriceNow(address tokenAddress) public payable returns(uint256 ethAmount, uint256 erc20Amount, uint256 blockNum)
```

#### Checking history price
input parameter | description 
---|---
tokenAddress | the erc20 address for checking 
num | the amount of prices 

output parameter | description 
---|---
uint256[] memory | price array 

The length of returned array is 3 * num，a price data consist of 3 numbers, respectively the eth amount, the erc20 amount and the effective block number.

```
function updateAndCheckPriceList(address tokenAddress, uint256 num) public payable returns (uint256[] memory)
```


#### Checking the least fee(ETH) for obtaining prices
input parameter | description 
---|---
tokenAddress | the erc20 address for checking 

output parameter | description 
---|---
--- | the least eth fee for obtaining prices 

The charging standard from this method is for ‘Checking price" and "Checking history price".

```
function checkPriceCostLeast(address tokenAddress) external view returns(uint256)
```

#### Checking the largest fee(ETH) for obtaining prices
input parameter | description 
---|---
tokenAddress | the erc20 address for checking 

output parameter | description 
---|---
--- | the largest eth fee for obtaining prices 

```
function checkPriceCostMost(address tokenAddress) external view returns(uint256)
```

#### Checking the fee(ETH) for obtaining a single price
input parameter | description 
---|---
tokenAddress | the erc20 address for checking 

output parameter | description 
---|---
--- | the eth fee for obtaining a single price 

```
function checkPriceCostSingle(address tokenAddress) external view returns(uint256)
```

### Demo
```
pragma solidity 0.6.0;

contract Nest_3_GetPrice {
    using SafeMath for uint256;
    using SafeERC20 for ERC20;
    
    // 投票合约
    Nest_3_VoteFactory _voteFactory;
    
    event price(uint256 ethAmount, uint256 tokenAmount, uint256 blockNum, uint256 ethMultiple, uint256 tokenForEth);
    event averagePrice(uint256 price);
    
    /**
     * @dev 初始化方法
     * @param voteFactory 投票合约
     */
    constructor (address voteFactory) public {
        _voteFactory = Nest_3_VoteFactory(address(voteFactory));
    }
    
    /**
     * @dev 获取单条价格
     * @param token 获取价格的 token 地址
     */
    function getSinglePrice(address token) public payable {
        // 考虑到未来升级，不排除升级价格合约的可能，必须使用投票合约查询价格合约地址。
        Nest_3_OfferPrice _offerPrice = Nest_3_OfferPrice(address(_voteFactory.checkAddress("nest.v3.offerPrice")));
        // 请求最新价格，返回 eth数量、token数量、生效价格区块号。费用暂定
        (uint256 ethAmount, uint256 tokenAmount, uint256 blockNum) = _offerPrice.updateAndCheckPriceNow.value(0.001 ether)(token);
        uint256 ethMultiple = ethAmount.div(1 ether);
        uint256 tokenForEth = tokenAmount.div(ethMultiple);
        // 如果支付价格的 eth 有剩余，需要处理
        // ........
        
        emit price(ethAmount, tokenAmount, blockNum, ethMultiple, tokenForEth);
    }
    
    /**
     * @dev 获取多条价格
     * @param token 获取价格的 token 地址
     * @param priceNum 获取价格的数量，从最新价格开始往后依次排序
     */
    function getBatchPrice(address token, uint256 priceNum) public payable {
        // 考虑到未来升级，不排除升级价格合约的可能，必须使用投票合约查询价格合约地址。
        Nest_3_OfferPrice _offerPrice = Nest_3_OfferPrice(address(_voteFactory.checkAddress("nest.v3.offerPrice")));
        /**
         * 返回的数组是3的整数倍，3个数据为一个价格数据
         * 分别对应，eth数量、token数量、生效价格区块号
         */
        uint256[] memory priceData = _offerPrice.updateAndCheckPriceList.value(0.01 ether)(token, priceNum);
        // 处理数据
        uint256 allTokenForEth = 0;
        uint256 priceDataNum = priceData.length.div(3);
        for (uint256 i = 0; i < priceData.length;) {
            uint256 ethMultiple = priceData[i].div(1 ether);
            uint256 tokenForEth = priceData[i.add(1)].div(ethMultiple);
            allTokenForEth = allTokenForEth.add(tokenForEth);
            i = i.add(3);
        }
        // 平均价格
        uint256 calculationPrice = allTokenForEth.div(priceDataNum);
        // 如果支付价格的 eth 有剩余，需要处理
        // ........
        
        
        emit averagePrice(calculationPrice);
    }
    
    /**
     * @dev 激活
     * @param nestAddress nestToken地址
     * @param nestAmount 激活销毁 Nest 数量
     */
    function activation(address nestAddress, uint256 nestAmount) public {
        // 考虑到未来升级，不排除升级价格合约的可能，必须使用投票合约查询价格合约地址。
        Nest_3_OfferPrice _offerPrice = Nest_3_OfferPrice(address(_voteFactory.checkAddress("nest.v3.offerPrice")));
        // 向价格合约授权 Nest，暂定数量为1万 
        ERC20(nestAddress).safeApprove(address(_offerPrice), nestAmount);
        // 激活
        _offerPrice.activation();
    }
    
    // 接收 eth 方法，必须实现
    receive() external payable {
        
    }
    
}

// 投票合约
interface Nest_3_VoteFactory {
    // 查询地址
    function checkAddress(string calldata name) external view returns (address contractAddress);
}

// 价格合约
interface Nest_3_OfferPrice {
    /**
    * @dev 激活
    */
    function activation() external;
    
    /**
    * @dev 查看最新价格
    * @param tokenAddress erc20地址 
    * @return ethAmount eth数量
    * @return erc20Amount erc20数量
    * @return blockNum 生效价格区块号
    */
    function updateAndCheckPriceNow(address tokenAddress) external payable returns(uint256 ethAmount, uint256 erc20Amount, uint256 blockNum);
    /**
    * @dev 查看生效价格列表
    * @param tokenAddress erc20地址
    * @param num 查询条数
    * @return uint256[] 价格列表
    */
    function updateAndCheckPriceList(address tokenAddress, uint256 num) external payable returns (uint256[] memory);
    /**
    * @dev 查看获取价格eth最少费用
    * @param tokenAddress erc20地址
    * @return uint256 获取价格eth最少费用 
    */
    function checkPriceCostLeast(address tokenAddress) external view returns(uint256);
    /**
    * @dev 查看获取价格eth最多费用 
    * @param tokenAddress erc20地址
    * @return uint256 获取价格eth最多费用 
    */
    function checkPriceCostMost(address tokenAddress) external view returns(uint256);
    /**
    * @dev 查看获取价格eth单条数据费用
    * @param tokenAddress erc20地址
    * @return uint256 获取价格eth单条数据费用
    */
    function checkPriceCostSingle(address tokenAddress) external view returns(uint256);
}


library SafeERC20 {
    using SafeMath for uint256;
    using Address for address;

    function safeTransfer(ERC20 token, address to, uint256 value) internal {
        callOptionalReturn(token, abi.encodeWithSelector(token.transfer.selector, to, value));
    }

    function safeTransferFrom(ERC20 token, address from, address to, uint256 value) internal {
        callOptionalReturn(token, abi.encodeWithSelector(token.transferFrom.selector, from, to, value));
    }

    function safeApprove(ERC20 token, address spender, uint256 value) internal {
        require((value == 0) || (token.allowance(address(this), spender) == 0),
            "SafeERC20: approve from non-zero to non-zero allowance"
        );
        callOptionalReturn(token, abi.encodeWithSelector(token.approve.selector, spender, value));
    }

    function safeIncreaseAllowance(ERC20 token, address spender, uint256 value) internal {
        uint256 newAllowance = token.allowance(address(this), spender).add(value);
        callOptionalReturn(token, abi.encodeWithSelector(token.approve.selector, spender, newAllowance));
    }

    function safeDecreaseAllowance(ERC20 token, address spender, uint256 value) internal {
        uint256 newAllowance = token.allowance(address(this), spender).sub(value, "SafeERC20: decreased allowance below zero");
        callOptionalReturn(token, abi.encodeWithSelector(token.approve.selector, spender, newAllowance));
    }
    function callOptionalReturn(ERC20 token, bytes memory data) private {
        require(address(token).isContract(), "SafeERC20: call to non-contract");
        (bool success, bytes memory returndata) = address(token).call(data);
        require(success, "SafeERC20: low-level call failed");
        if (returndata.length > 0) {
            require(abi.decode(returndata, (bool)), "SafeERC20: ERC20 operation did not succeed");
        }
    }
}

interface ERC20 {
    function totalSupply() external view returns (uint256);
    function balanceOf(address account) external view returns (uint256);
    function transfer(address recipient, uint256 amount) external returns (bool);
    function allowance(address owner, address spender) external view returns (uint256);
    function approve(address spender, uint256 amount) external returns (bool);
    function transferFrom(address sender, address recipient, uint256 amount) external returns (bool);
    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
}

library SafeMath {
    function add(uint256 a, uint256 b) internal pure returns (uint256) {
        uint256 c = a + b;
        require(c >= a, "SafeMath: addition overflow");

        return c;
    }
    function sub(uint256 a, uint256 b) internal pure returns (uint256) {
        return sub(a, b, "SafeMath: subtraction overflow");
    }
    function sub(uint256 a, uint256 b, string memory errorMessage) internal pure returns (uint256) {
        require(b <= a, errorMessage);
        uint256 c = a - b;

        return c;
    }
    function mul(uint256 a, uint256 b) internal pure returns (uint256) {
        if (a == 0) {
            return 0;
        }
        uint256 c = a * b;
        require(c / a == b, "SafeMath: multiplication overflow");

        return c;
    }
    function div(uint256 a, uint256 b) internal pure returns (uint256) {
        return div(a, b, "SafeMath: division by zero");
    }
    function div(uint256 a, uint256 b, string memory errorMessage) internal pure returns (uint256) {
        require(b > 0, errorMessage);
        uint256 c = a / b;
        return c;
    }
    function mod(uint256 a, uint256 b) internal pure returns (uint256) {
        return mod(a, b, "SafeMath: modulo by zero");
    }
    function mod(uint256 a, uint256 b, string memory errorMessage) internal pure returns (uint256) {
        require(b != 0, errorMessage);
        return a % b;
    }
}

library Address {
    function isContract(address account) internal view returns (bool) {
        bytes32 codehash;
        bytes32 accountHash = 0xc5d2460186f7233c927e7db2dcc703c0e500b653ca82273b7bfad8045d85a470;
        assembly { codehash := extcodehash(account) }
        return (codehash != accountHash && codehash != 0x0);
    }
    function sendValue(address payable recipient, uint256 amount) internal {
        require(address(this).balance >= amount, "Address: insufficient balance");
        (bool success, ) = recipient.call.value(amount)("");
        require(success, "Address: unable to send value, recipient may have reverted");
    }
}
```
