# limeacademy

## BlockChain 101

### Intro to Blockchain

- What is a transaction?

  A transaction is the way we interact with the blockchain. Transactions can change the blockchain data ( like wallet balances, smart contracts, etc...).

- What is the approximate time of every Ethereum transaction?

  It can go from few seconds to minutes depending on the gas and network load

- What is a node?

  A node is a "computer" that is part of a group ( of nodes ) that validates and sync the status of a blocakchain by reaching a consensus with all other nodes.

## Solidity & Smart Contracts

### Learning Solidity

- What is delegatecall? Give example with code.

  DelegateCalls is a feature that allows a contract to call a function in another contract preserving the values from the caller

  - address
  - msg.sender
  - msg.value

  In reality, when using delegatecall the function code from target contract is execute on the caller contract, meaning, all read / write to variables is done in the caller contract.

  ```
  contract A {
    string internal name = "name_a";

    function test(address contractB, string memory newName) external {
      (bool successSet, ) = contractB.delegatecall(
        abi.encodeWithSelector(B(contractB).testSetDelegateCall.selector, newName)
      );
      require(successSet, "error setting name in delegate call");
    }

    function read() public view returns (string memory) {
      return name;
    }
  }

  contract B {
    string internal name = "name_b";

    function testSetDelegateCall(string calldata _newName) external {
      name = _newName;
    }
  }
  ```

- What is multicall? Give example with code.

  multicall is used to batch multiple functions requests in a single function call.
  example from https://github.com/indexed-finance/multicall/blob/master/contracts/MultiCall.sol

  ```
  contract MultiCall {
    constructor(
      address[] memory targets,
      bytes[] memory datas
    ) public {
      uint256 len = targets.length;
      require(datas.length == len, "Error: Array lengths do not match.");

      bytes[] memory returnDatas = new bytes[](len);

      for (uint256 i = 0; i < len; i++) {
        address target = targets[i];
        bytes memory data = datas[i];
        (bool success, bytes memory returnData) = target.call(data);
        if (!success) {
          returnDatas[i] = bytes("");
        } else {
          returnDatas[i] = returnData;
        }
      }
      bytes memory data = abi.encode(block.number, returnDatas);
      assembly { return(add(data, 32), data) }
    }
  }
  ```

- What is time lock? Give example with code.

  Time lock is a pattern used to delay the execution of a transaction by a time duration. One way to achieve this, is by computing an identifier of the transaction and storing it in the blockchain with the current block.timestamp. After the timelock passes, the transaction that computes to the same txId can be executed.

  ```
  contract TimeLock {
    error NotOwnerError();
    error AlreadyQueuedError(bytes32 txId);
    error TimestampNotInRangeError(uint blockTimeStamp, uint timestamp);
    error NotQueuedError(bytes32 txId);
    error TimestampNotPassedError(uint blockTimeStamp, uint timestamp);
    error TimestampExpiredError(uint blockTimeStamp, uint timestamp);
    error TxFailError();

    event Queue(
        bytes32 indexed txId,
        address indexed target,
        uint value,
        string func,
        bytes data,
        uint _timestamp
    );

    event Executed(
        bytes32 indexed txId,
        address indexed target,
        uint value,
        string func,
        bytes data,
        uint _timestamp
    );

    event Cancel(
        bytes32 indexed txId
    );

    uint public constant MIN_DELAY = 10;
    uint public constant MAX_DELAY = 1000;
    uint public constant GRACE_PERIOD = 1000;

    address public owner;
    mapping(bytes32 => bool) public queued;

    constructor(){
        owner = msg.sender;
    }

    receive() external payable {}

    modifier onlyOwner(){
        if (msg.sender != owner){
            revert NotOwnerError();
        }
        _;
    }

    function getTxId(
        address _target,
        uint _value,
        string calldata _func,
        bytes calldata _data,
        uint _timestamp
    ) public pure returns (bytes32 txId){
        return keccak256(
            abi.encode(
                _target, _value, _func, _data, _timestamp
            )
        );
    }

    function queue(
        address _target,
        uint _value,
        string calldata _func,
        bytes calldata _data,
        uint _timestamp
    ) external onlyOwner {
        bytes32 txId = getTxId(_target, _value, _func, _data, _timestamp);
        if (queued[txId]){
            revert AlreadyQueuedError(txId);
        }
        if (
            _timestamp < block.timestamp + MIN_DELAY ||
            _timestamp > block.timestamp + MAX_DELAY
        ){
            revert TimestampNotInRangeError(block.timestamp, _timestamp);
        }
        queued[txId] = true;
        emit Queue(txId, _target,_value,_func, _data, _timestamp);
    }


    function execute(
        address _target,
        uint _value,
        string calldata _func,
        bytes calldata _data,
        uint _timestamp
    ) external payable onlyOwner returns (bytes memory) {
        bytes32 txId = getTxId(_target, _value, _func, _data, _timestamp);
        if (!queued[txId]){
            revert NotQueuedError(txId);
        }
        if (block.timestamp < _timestamp ){
            revert TimestampNotPassedError(block.timestamp, _timestamp);
        }
        if (block.timestamp > _timestamp + GRACE_PERIOD ){
            revert TimestampExpiredError(block.timestamp, _timestamp + GRACE_PERIOD);
        }
        queued[txId] = false;

        bytes memory data;
        if (bytes(_func).length > 0){
            data = abi.encodePacked(
                bytes4(keccak256(bytes(_func))), _data
            );
        }else{
            data = _data;
        }

        (bool ok, bytes memory res) = _target.call{value: _value}(_data);
        if (!ok){
            revert TxFailError();
        }
        emit Executed(txId, _target, _value, _func,_data, _timestamp);
        return res;
    }

    function cancel(bytes32 _txId) external onlyOwner {
        if (!queued[_txId]){
            revert NotQueuedError(_txId);
        }
        queued[_txId] = false;
        emit Cancel(_txId);
    }
  }
  ```
