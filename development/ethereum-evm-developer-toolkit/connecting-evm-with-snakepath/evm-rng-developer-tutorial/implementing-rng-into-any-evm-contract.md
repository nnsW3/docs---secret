# Implementing RNG into any EVM Contract

After we've gone through an extensive example on how our example contract works, here's how to implement SecretVRF via Snakepath in your own contract in 4 easy steps:&#x20;

## Importing the Interface

First, import the `ISecretVRF` interface into your Solidity Contract:&#x20;

```solidity
/// @notice Interface of the VRF Gateway contract. Must be imported. 
interface ISecretVRF { 
    function requestRandomness(uint32 _numWords, uint32 _callbackGasLimit) external payable returns (uint256 requestId); 
}
```

## Set the SecretVRF gateway contract&#x20;

Second, set your gateway address to the Secret VRF Gateways that you can find [evm-testnet](../../connecting-evm-with-snakepathrng/gateway-contracts/evm-testnet/ "mention") and [evm-mainnet](../../connecting-evm-with-snakepathrng/gateway-contracts/evm-mainnet/ "mention"). You only need to make sure that your contract knows the correct SecretVRF Gateway address, for example:

```solidity
/// @notice VRFGateway stores address to the Gateway contract to call for VRF
address public VRFGateway;

address public immutable owner;

constructor() {
   owner = msg.sender;
}

modifier onlyOwner() {
    require(msg.sender == owner, "UNAUTHORIZED");
    _;
}

/// @notice Sets the address to the Gateway contract 
/// @param _VRFGateway address of the gateway
function setGatewayAddress(address _VRFGateway) external onlyOwner {
    VRFGateway = _VRFGateway;
}
```

## Call the SecretVRF Gateway contract

Now, we implement the function that calls the SecretVRF Gateway on EVM. Note that you _**have**_ to pay an extra amount of your gas token as **CallbackGas:**

```solidity
/// @notice Event that is emitted when a VRF call was made (optional) 
/// @param requestId requestId of the VRF request. Contract can track a VRF call that way 
event requestedRandomness(uint256 requestId);

/// @notice Demo function on how to implement a VRF call using Secret VRF
function requestRandomnessTest(uint32 _numWords, uint32 _callbackGasLimit) external payable {
    // Get the VRFGateway contract interface 
    ISecretVRF vrfContract = ISecretVRF(VRFGateway);

    // Call the VRF contract to request random numbers. 
    // Returns requestId of the VRF request. A  contract can track a VRF call that way.
    uint256 requestId = vrfContract.requestRandomness{value: msg.value}(_numWords, _callbackGasLimit);

    // Emit the event
    emit requestedRandomness(requestId);
}
```

The callback gas is the amount of gas that you have to pay for the message coming on the way back. If you do pay less than the amount speficified below, your Gateway TX will fail:&#x20;

<pre class="language-solidity"><code class="lang-solidity">/// @notice Increase the task_id to check for problems 
/// @param _callbackGasLimit the Callback Gas Limit

function estimateRequestPrice(uint32 _callbackGasLimit) private view returns (uint256) {
<strong>    uint256 baseFee = _callbackGasLimit*block.basefee;
</strong>    return baseFee;
}

</code></pre>

Since this check is dependent on the current block.basefee of the block it is included, it is recommended that you estimate the gas fee beforehand and add some extra overhead. An example of how this can be implemented can be found here.&#x20;

## Wait for the callback

From here, the SecretVRF Gateway will take care of everything, just wait 1-2 blocks for Gateway to provide you the random number by getting it from the Secret Networks on chain VRF and do the callback.

The SecretVRF gateway contract will _always_ call the contract that called the VRF contract (using `msg.sender`) with the function selector bytes `0x38ba4614`, which is the function:

```solidity
function fulfillRandomWords(uint256 requestId, uint256[] calldata randomWords) external 
```

Now, the SecretVRF Gateway contract will verify the validity of the call and when all checks pass, it will call this function. In this case, we just emit a log as an example to finish our demo. Emitting a log is _**not obligatory and optional.**_

```solidity
event fulfilledRandomWords(uint256 requestId, uint256[] randomWords);
/// @notice Callback by the Secret VRF with the requested random numbers
/// @param requestId requestId of the VRF request that was initally called
/// @param randomWords Generated Random Numbers in uint256 array
function fulfillRandomWords(uint256 requestId, uint256[] calldata randomWords) external {
    // Checks if the callback was called by the VRFGateway and not by any other address
    require(msg.sender == address(VRFGateway), "only Secret Gateway can fulfill");

    // Do your custom stuff here, for example:
    emit fulfilledRandomWords(requestId, randomWords);
}
```