# TokenSwapWith0x
TokenSwapWith0x
Integrating 0x into your contracts

We are going to add automatic conversion of ETH to DAI to our contracts. It's always best to learn by example, so let's try to do this for a contract. A fully working Remix example can be found at the end.
1. Adding automatic conversions

Assume we have a pay function that users can call to interact with our contract. We want to give them the option to pay in DAI directly or they can automatically trade ETH to DAI using the 0x API.

But now we want to add a second option for the user to pay in ETH. You can see this on the right. In the case of users sending 0 ETH (msg.value == 0), we assume that the user wants to pay in DAI directly. Then we do a transferFrom as before and additionally require all swap related parameters to be empty.

In the case of a user wanting to trade his ETH, we call our convert function _convertEthToDai with all the swap parameters. Let's take a look how the convert function looks like...

function pay(
    uint256 paymentAmountInDai,
    address spender,
    address payable swapTarget,
    bytes calldata swapCallData
) public payable {
    if (msg.value > 0) {
        _convertEthToDai(
            paymentAmountInDai,
            spender,
            swapTarget,
            swapCallData
        );
    } else {
        require(
            spender == address(0),
            "EMPTY_SPENDER_WITHOUT_SWAP"
        );
        require(
            swapTarget == address(0),
            "EMPTY_TARGET_WITHOUT_SWAP"
        );
        require(
            swapCallData.length == 0,
            "EMPTY_CALLDATA_WITHOUT_SWAP"
        );
        require(
            DAI.transferFrom(msg.sender, address(this), paymentAmountInDai),
            "DAI transfer failed"
        );
    }

    // do something with that DAI    
    [...]
}

2. Converting the ETH to DAI

Now we get to the heart of the logic. On a high level we do

    wrap ETH into WETH
    approve WETH for target
    execute 0x API swap
    run refunds

How you would get these parameters for the conversion, you can see in step 3. But essentially what they represent is an optimized swap contract call. swapTarget.call(swapCallData) executes the trade which internally transfers the WETH funds from this contract to the spender address.

Once the swap is finished, we can return any leftover ETH and DAI to the trader. The DAI refund might not be required as the leftover amount in my tests was always almost non existent. But keep it when doing large scale trades to be safe.

3. Retrieving the API Request Data

Now we have the swap functionality, but the user still needs to know what to send as the swap parameters. For this we add a view function that returns the 0x API Request URL. We use https://kovan.api.0x.org for our tests, for mainnet you of course would use https://api.0x.org/.

For example given we want to buy one DAI, we would pass "1000000000000000000" (1e18) to the function and we get our request url looking like this: https://kovan.api.0x.org/swap/v1/quote?sellToken=0xd0A1E359811322d97991E03f863a0C30C2cF029C&buyToken=0x1528f3fcc26d13f7079325fb78d9442607781c8c&buyAmount=1000000000000000000. You can directly open this in the browser to get the results. Obviously if you have a frontend, you can automate all of this.

Now once you click the link, all you need are three values:

    "allowanceTarget" → address spender
    "to" → address payable swapTarget
    "data" → bytes calldata swapCallData

string private api0xUrl = 'https://kovan.api.0x.org/swap/v1/quote';
string private wethToDai0xApiRequest = '?sellToken=0xd0A1E359811322d97991E03f863a0C30C2cF029C&buyToken=0x1528f3fcc26d13f7079325fb78d9442607781c8c&buyAmount=';

function get0xApiRequest(uint256 paymentAmountInDai) external view returns(string memory) {
    return string(bytes(api0xUrl).concat(bytes(wethToDai0xApiRequest)).concat(paymentAmountInDai.toBytes()));
}

Now once you click the link, all you need are three values:

    "allowanceTarget" → address spender
    "to" → address swapTarget
    "data" → bytes calldata swapCallData

With these variables a user now has all he needs to call  myContract.pay(amount, spender, swapTarget, swapCallData).
4. Useful Helper Functions
Helper Ralph

If you wondered about some of the functions in the code above, don't worry. We've implemented and used a few useful helper functions, namingly concat, toStringBytes and getRevertMsg. Those may be quite useful to you in general, so it's worth taking a closer look.
1. Concat String Bytes

With the new abi.encodePacked function since Solidity v5, concatenating strings is particularly easy. You can use this function for strings like this: concat(bytes(myString1), bytes(myString2)).

function concat(
    bytes memory a,
    bytes memory b
) internal pure returns (bytes memory) {
    return abi.encodePacked(a, b);
}

2. Uint256 to String Bytes

Inspired by the Provable code here, this function computes the string representation of a uint256 number returned as bytes array.

Strings in Solidity are UTF-8 encoded. The value 48 implies the character '0'. So to convert a number to the correct string, we essentially compute and store 48 + remainder of modulo 10 for each digit.

function toStringBytes(
     uint256 v
) internal pure returns (bytes memory) {
    if (v == 0) { return "0"; }

    uint256 j = v;
    uint256 len;

    while (j != 0) {
        len++;
        j /= 10;
    }

    bytes memory bstr = new bytes(len);
    uint256 k = len - 1;
    
    while (v != 0) {
        bstr[k--] = byte(uint8(48 + v % 10));
        v /= 10;
    }
    
    return bstr;
}

3. Get Revert Message for Low-level Call

Lastly we added a function to retrieve the revert message from low-level contract calls. This allows us to give more detailed information about the revert reason. Implementation was taken from Stackexchange, a website you should check out whenever you can.

function getRevertMsg(
    bytes memory _returnData
) internal pure returns (string memory) {
    if (_returnData.length < 68)
        return 'Transaction reverted silently';

    assembly {
        _returnData := add(_returnData, 0x04)
    }

    return abi.decode(_returnData, (string));
}
