1. The OZ upgrade tool for hardhat defends against 6 kinds of mistakes. What are they and why do they matter?
    1. No `constructor()`. This cannot be applied to proxy. It has to used `initialize()`. The reason why `constructor()` not working is that the constructor is called only once during the deployment of the implement contract. But the interaction between Proxy and implement contracts is done by `delegatecall` which can't call the constructor. Therefore any code in constructor won't be applied to the proxy contract. However `constant` and `immutable` can be used in constructor exceptionally.

        EX: 
        ```solidity
            uint public immutable X;

            /// @custom:oz-upgrades-unsafe-allow not valid for contracts
            constructor(uint _x) {
                X = _x; // This works
            }
        ```
    1. No calling `initialize()` more than once.
    1. No storage variable with an initial value.

        EX 1: `uint public x = 100; // (X)`

        EX 2: `uint public; // (O)`

        EX 3: `uint public constant x = 100; // (O)`
        
        EX 4: `uint public immutable x = 100; // (O)`

        `constant` and `immutable` are compiled into contract byte code. Therefore it's possible to set a value.
    1. Don't reorder variables when you upgrade a contract (adding new under variable is OK). Changing order of variable means changing storage layout. It will cause unexpected result after `delegatecall`.
    1. Proxy contract and Implementation contract both should have the same set of variables.
    1. No `seltdestruct()`.

1. What is a beacon proxy used for?

    Useful when there are multiple proxies pointing to one single implementation contract.
    For example, let's say there are 10 proxies pointing to a implementation contract called X. And it's using Transparent pattern which each proxy holds the X's address as variable. When there is a new version of X (let's call it X2), then the 10 proxies should update its variables as X2's address which is cumbersome. Beacon pattern can make things easier by creating a new contract that holds address. The process is going to be like below:
        
        1. Call to a proxy
        1. proxy calls to beacon
        1. beacon returns implementation's address to proxy
        1. proxy calls to implementation by delegate call

    It's the same as transparent pattern that storage variables are stored in each proxy contract.

1. Why does the openzeppelin upgradeable tool insert something like __gap[48] inside the contracts? To see it, create an upgradeable smart contract that has a parent contract and look in the parent.
    It's to preserve storage space from parent contract. Otherwise there is risk of stroage crash. If there is no such `__gap[48]`, the potential problem can occur during upgrading proxy contract. 

    ```solidity
    contract Base {
        uint256 base1;      // storage layout 0
    }

    contract Child is Base {
        uint256 child;      // storage layout 1
    }
    ```

    When it is complied, it will become one single bytecode. And it will be deployed and interact with Proxy contract and Proxy contract has the same storage variables.

    Then let's say there is a new version of `Base` contract.

    ```solidity
    contract Base {
        uint256 base1;      // storage layout 0
        uint256 base2;      // storage layout 1 <- note: a new variable added
    }

    contract Child is Base {
        uint256 child;      // storage layout 2
    }
    ```

    Imagine this is deployed. `child` is no longer in in slot 1 but 2. Then on the proxy contract, Then the previous `child` will be gone because it's overriden by `base2`.
    
    To prevent this, it should be like:
    ```solidity
    contract Base {
        uint256 base1;      // storage layout 0
        uint256[49] __gap;  // storage layout 1 - 50
    }

    contract Child is Base {
        uint256 child;      // storage layout 51
    }
    ```
    Then when add a new variable:
     ```solidity
    contract Base {
        uint256 base1;      // storage layout 0
        uint256 base2;      // storage layout 1
        uint256[48] __gap;  // storage layout 2 - 50 <- note: 1 reduced
    }

    contract Child is Base {
        uint256 child;      // storage layout 51 <- note: still 51
    }
    ```


1. What is the difference between initializing the proxy and initializing the implementation? Do you need to do both? When do they need to be done?
    1. "What is the difference between initializing the proxy and initializing the implementation?"

        There is no specific things to do for initializing the proxy.
        On the other hand, the implementation contract needs a specific initializing process. Look at the example below:
        
        ```solidity
        contract myImplContract {
            uint256 x;

            function initialize(uint256 _x) {
                x = _x;
            }
        }
        ```

        It's like `constructor` but the `constructor` cannot be applicable for the proxy pattern since we use `deligatecall` from the proxy contract (deligatecall cannot call constructor).

        Thus we need constructor but not constructor ;)
        We should use just a normal custom function and it can be like the above `initialize`. But to be like constructor, it should be called only once. To do that, we can play with a `bool` variable.

        OZ provides libraries for upgradeable contracts.

    1. "Do you need to do both? When do they need to be done?"
        
        Both are needed in order to implemented upgradeable contract.



1. What is the use for the [reinitializer](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/master/contracts/proxy/utils/Initializable.sol#L119)? Provide a minimal example of proper use in Solidity

    `reinitializer` is used to for subsequent upgradeable contract.

    1. Develop a new version of implementation contract ( logic change &version update )
    1. Develop a new version of implementation contract and introduce a new storage variables.

    ```solidity
    // let's say there is already deployed contract V1.
    contract MyImplContract {
        function initializeV2() reinitializer(2) {  //<- 2 is for new version
            // introduce new logic or assign variable value here
        }
    }

    ```