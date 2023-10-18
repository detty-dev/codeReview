### CURVELPORACLE

### INTRODUCTION

    CurveLPOracle is a tool used within the Maker ecosystem to accurately calculate the price of Liquidity Provider tokens(LP) in curve pools. Its main purpose is to determine the price of LP tokens, which represent liquidity in a curve pool, by combining information from the curve pool's get_virtual_price() function and the lowest rate among the underlying assets returned by one of the stored oracles.

    Breakdown of how CurveLPOracle operates:

    Price Calculation: The price of the LP token is determined by multiplying the virtual price obtained from the get_virtual_price() function of the curve pool by the lowest rate among the underlying assets. This calculation results in a lower bound evaluation of the LP token's value.

    Curve Pools: Curve pools are a specific type of Automated Market Maker (AMM) pools. They are designed to use liquidity more effectively by concentrating most of the liquidity at current prices.

    Feed Variables: CurveLPOracle operates with two feed variables: cur and nxt. These variables store the current price (cur) and the queued price (nxt). The prices propagate through the system with a 1-hour delay. This delay allows authorized parties (wards) to take measures in case the queued price (nxt) is set to an incorrect value.

### Functionalities:

   * stop(): This function can only be called by authorized wards and is used to stop the oracle. Stopping the oracle means it will not provide price information until it's started again.

    * start(): Similar to the stop() function, start() can only be called by authorized wards, and it is used to remove the stop flag, allowing the oracle to resume providing price information.

    * step(): Again, this function can only be called by authorized wards and is used to update the hop value, which is set to a default of 1 hour. The hop value likely represents the propagation delay mentioned earlier.

    * link(): This function, too, can only be called by authorized wards. It is used to update the oracle address, which might be necessary when the oracle's location or configuration needs to be changed.

    * zzz(): This function can be called by anyone and returns the timestamp of the last price update.

    * pass(): Also callable by anyone, it returns true if enough time has passed since the last update to compute a new price.

    * poke(): Again, anyone can call this function if pass() returns true, and it calculates the new price of the LP token.

    * peek(): Accessible only to whitelisted addresses in the bud mapping, it returns the current price and its validity.

    * peep(): Like peek(), this function is available only to whitelisted addresses in the bud mapping. It returns the queued price, which will become the current price in the next call of poke(), along with its validity.

    * read(): Exclusive to whitelisted addresses in the bud mapping, it returns the current price as a bytes32 value.

    * kiss(): Authorized wards can use this function to add one or multiple addresses to the whitelist in the bud mapping.

    * diss(): Authorized wards can remove one or multiple addresses from the whitelist in the bud mapping.

    * rely() and deny(): These are standard authorization functions.

### iNTERFACES;: 

        interface AddressProviderLike {
        function get_registry() external view returns (address);
}   

       interface CurveRegistryLike {
       function get_n_coins(address) external view returns (uint256[2] calldata);
}

       interface CurvePoolLike {
       function coins(uint256) external view returns (address);
       function get_virtual_price() external view returns (uint256);
       function lp_token() external view returns (address);
       function remove_liquidity(uint256, uint256[2] calldata) external returns (uint256);
}

       interface OracleLike {
       function read() external view returns (uint256);
}

## CurveLPOracleFactory:

    it serves as a factory for creating instances of CurveLPOracle contracts  by calling the build() function, allowing users to set various parameters and interact with the curve pool to monitor and manage prices for tokens within that pool. The contract leverages an address provider to fetch relevant addresses from other contracts in the ecosystem.

### contract CurveLPOracleFactory {

    AddressProviderLike public immutable ADDRESS_PROVIDER;

    event NewCurveLPOracle(address owner, address orcl, bytes32 wat, address pool, bool nonreentrant);

    constructor(address addressProvider) {
        ADDRESS_PROVIDER = AddressProviderLike(addressProvider);
    }

    function build(
        address _owner,
        address _pool,
        bytes32 _wat,
        address[] calldata _orbs,
        bool _nonreentrant
    ) external returns (address orcl) {
        uint256 ncoins = CurveRegistryLike(ADDRESS_PROVIDER.get_registry()).get_n_coins(_pool)[1];
        require(ncoins == _orbs.length, "CurveLPOracleFactory/wrong-num-of-orbs");
        orcl = address(new CurveLPOracle(_owner, _pool, _wat, _orbs, _nonreentrant));
        emit NewCurveLPOracle(_owner, orcl, _wat, _pool, _nonreentrant);
    }
}

1. AddressProviderLike: The contract starts by referencing an interface named AddressProviderLike. This interface likely defines functions that the contract needs to interact with.

2. ADDRESS_PROVIDER: its a state variable declared as public immutable of type AddressProviderLike. its main purpose is to store an address that points to an "address provider" contract. This address provider is used to retrieve the registry address later in the contract.

3. event NewCurveLPOracle: it is an event that is emitted when a new CurveLPOracle contract is created. It logs various details, including the owner's address, the oracle's address, a label (bytes32), the pool address, and a boolean flag that indicates whether the operation is non-reentrant (i.e., whether it can be repeated).

4. constructor: The constructor function takes an address as a parameter, which is intended to be the address of an "address provider" contract. It initializes the ADDRESS_PROVIDER state variable with this address.

### Function build: 
   This is the core function of the contract, and it is intended to create new instances of the CurveLPOracle contract. It takes the following parameters:

    address_owner: The address of the owner of the new CurveLPOracle contract.

    address_pool: The address of the curve pool associated with this price feed.

    bytes32_wat: A label (in bytes32 format) that represents the token whose price is being tracked.

    address[] calldata _orbs: An array of addresses representing oracles for the tokens in the pool.

    bool _nonreentrant: A boolean flag indicating whether the operation is non-reentrant.
    Inside the function:

### Function Logic: 
   1. It first calculates the number of coins (tokens) in the curve pool by calling the get_n_coins() function from a contract referenced by ADDRESS_PROVIDER. It uses [1] to get the second value returned by this function.

   2. The require condition checks if the number of tokens in the curve pool (ncoins) is equal to the length of the _orbs array. If they are not equal, it raises an error.

   3. It creates a new instance of the CurveLPOracle contract, passing in the provided parameters.

   4. It emits the NewCurveLPOracle event to log the creation of the new oracle.

### CurveLPOracle:

    CurveLPOracle contract provides a mechanism for managing and updating the prices of LP tokens for a Curve pool. It includes functions for price updates, whitelisting, and admin control to ensure accurate and secure price calculations.

### contract CurveLPOracle {

    mapping (address => uint256) public wards;                                      
    function rely(address _usr) external auth { wards[_usr] = 1; emit Rely(_usr); }  // Add admin
    function deny(address _usr) external auth { wards[_usr] = 0; emit Deny(_usr); }  // Remove admin
    modifier auth {
        require(wards[msg.sender] == 1, "CurveLPOracle/not-authorized");
        _;
    }

   * The above contract code  maintains a mapping called wards that keeps track of addresses with administrative authority.

   * The rely function allows an address to be added as an admin, granting them certain privileges.
    
   * The deny function removes admin privileges from an address.
    
   * A modifier named auth is used to ensure that only authorized addresses (admins) can execute certain functions.

    
        address public immutable src; 
        uint8   public stopped;        
        uint16  public hop = 1 hours;  
        uint232 public zph; 

   * The code above stores information about the price source in the src variable.

   * It combines the stopped, hop, and zph parameters into a single storage slot to reduce storage costs.

   * stopped is a flag to start or stop updates.

   *  hop represents the minimum time between price updates, set to 1 hour by default.

   * ph stores the timestamp of the last price update plus the hop value.

       
# Whitelisting:

    mapping (address => uint256) public bud;
    modifier toll { require(bud[msg.sender] == 1, "CurveLPOracle/not-whitelisted"); _; }

    The contract maintains a mapping called bud to whitelist addresses.
    The toll modifier ensures that only whitelisted addresses can execute certain functions.

# Feed Structs:

    Two feed structures, cur and nxt, store the current and queued prices along with their validity status.

# Constructor:
        constructor(address _ward, address _pool, bytes32 _wat, address[] memory _orbs, bool _nonreentrant) {
        require(_pool != address(0), "CurveLPOracle/invalid-pool");
        uint256 _ncoins = _orbs.length;
        pool   = _pool;
        src    = CurvePoolLike(_pool).lp_token();
        wat    = _wat;
        ncoins = _ncoins;
        nonreentrant = _nonreentrant;
        
        require(_ward != address(0), "CurveLPOracle/ward-0");
        wards[_ward] = 1;
        emit Rely(_ward);
    }
1. The constructor initializes key state variables, sets up the configuration of the CurveLPOracle contract, and assigns an admin role to the specified address. It also ensures that the provided parameters are valid and that the contract is properly configured during deployment.

2. require(_pool != address(0), "CurveLPOracle/invalid-pool");: This line checks if the _pool address is not zero (i.e., a valid address). If it's zero, it raises an error indicating an invalid pool address.
Initialization of State Variables:

3. uint256 _ncoins = _orbs.length;: This line calculates the length of the _orbs array and stores it in the _ncoins variable.
   
4. pool = _pool;: This line initializes the pool state variable with the provided _pool address. This is the address of the underlying Curve pool.
   
5. src = CurvePoolLike(_pool).lp_token();: It initializes the src state variable with the LP token address (from the underlying Curve pool) that will serve as the price source.
   
6. wat = _wat;: The wat state variable is initialized with the provided label _wat, which represents the token whose price is being tracked.

7. ncoins = _ncoins;: The ncoins state variable is set to the calculated _ncoins, representing the number of tokens in the underlying Curve pool.
   
8. nonreentrant = _nonreentrant;: This variable is set based on the provided _nonreentrant boolean, which determines whether reentrancy protection is enabled.
Oracle Addresses:

9. The code iterates over the provided _orbs array and performs the following actions within a loop:
require(_orbs[i] != address(0), "CurveLPOracle/invalid-orb");: It checks if each address in the _orbs array is not zero (i.e., a valid address). If any of them are zero, it raises an error indicating an invalid oracle address.
        for (uint256 i = 0; i < _ncoins;) {
            require(_orbs[i] != address(0), "CurveLPOracle/invalid-orb");
            
orbs.push(_orbs[i]);: The valid oracle addresses are added to the orbs array, which represents the addresses of price feeds for the pool's assets.
        orbs.push(_orbs[i]);
            unchecked { i++; }
        }

# Admin Role Assignment:               
        require(_ward != address(0), "CurveLPOracle/ward-0");: This line checks if the provided _ward address (an admin address) is not zero.
          require(_ward != address(0), "CurveLPOracle/ward-0");m 

        wards[_ward] = 1;: It sets the admin address (the _ward) in the wards mapping to 1, indicating that this address has administrative privileges within the contract.
                    wards[_ward] = 1;

        emit Rely(_ward);: An event Rely is emitted to log the addition of the admin address.
                    emit Rely(_ward);
            }    

### Price Management Functions:
   
### function stop() external auth:
        function start() external auth {
        stopped = 0;
        emit Start();
    }
This function is publicly callable (external) and can be executed by an authorized address due to the auth modifier. The purpose of this function is to  halt price updates by setting the stopped flag to 1, clearing the current and queued price data, and resetting the last price update time to zero. This function is intended for scenarios where you want to temporarily pause price updates, possibly for maintenance or other reasons, while still allowing authorized addresses to resume updates when needed.

* stopped = 1;: This line sets the stopped state variable to 1. By setting stopped to 1, the contract indicates that price updates are stopped or disabled.

* delete cur; and delete nxt;: These lines delete the values in the cur and nxt variables, which represent the current and queued prices. Deleting these values effectively clears the stored price data.

* zph = 0;: This line sets the zph (time of last price update plus hop) to 0, effectively resetting the time of the last price update.

* emit Stop();: This line emits a Stop event to log that the price updates have been stopped. Events are typically used for logging important actions within a smart contract, allowing external parties to track the contract's activity.

### function start() external auth:    
        function start() external auth {
        stopped = 0;
        emit Start();
    }
This function is publicly callable (external) and can be executed by an authorized address due to the auth modifier. The purpose of this function resume or start the price updates after they have been temporarily stopped using the stop() function. It allows authorized addresses to re-enable the price update mechanism when necessary, providing flexibility and control over the price management process.

* stopped = 0;: This line sets the stopped state variable to 0. By setting stopped to 0, the contract indicates that price updates are no longer stopped, and the system is free to resume updating prices.

* emit Start();: This line emits a Start event to log that the price updates have been started or resumed. Events are typically used for logging important actions within a smart contract, allowing external parties to track the contract's activity.

### function step(uint16 _hop) external auth: 
        function step(uint16 _hop) external auth {
        uint16 old = hop;
        hop = _hop;
        if (zph > old) {  // if false, zph will be unset and no update is needed
            zph = (zph - old) + _hop;
        }
        emit Step(_hop);
    }
    This function is publicly callable (external) and can be executed by an authorized address due to the auth modifier. The purpose of this function is to allow an authorized address to adjust the time interval between price updates. It updates the hop value, recalculates the zph value to account for the time passed since the last update, and logs the change using an event. This functionality provides flexibility in controlling the frequency of price updates within the contract.

   * uint16 old = hop;: This line stores the current value of the hop state variable (the existing time interval) in a variable named old to keep track of the previous setting.

   * hop = _hop;: This line updates the hop state variable with the new time interval value provided as an argument to the function. In other words, it changes the minimum time between price updates to the new value specified in _hop.

  * if (zph > old) { ... }: This condition checks whether the current zph (time of the last price update plus the previous hop value) is greater than the previous hop value (old). If this condition is true, it means that there was an active time interval, and a price update was expected.

 * zph = (zph - old) + _hop;: If the condition in the previous step is true, this line calculates the new zph value. It subtracts the previous hop value (old) from the existing zph and then adds the new hop value (_hop). This calculation ensures that the new zph accounts for the time that has passed since the last update, plus the newly specified time interval.

 * emit Step(_hop);: This line emits a Step event to log the change in the hop value. Events are used for logging important actions within a smart contract.

### function link(uint256 _id, address _orb) external auth: 
        function link(uint256 _id, address _orb) external auth {
        require(_orb != address(0), "CurveLPOracle/invalid-orb");
        require(_id < ncoins, "CurveLPOracle/invalid-orb-index");
        orbs[_id] = _orb;
        emit Link(_id, _orb);
    }
    This function is publicly callable (external) and can be executed by an authorized address due to the auth modifier. The purpose of this function is to link or associate a new oracle address (_orb) with a specific underlying asset based on its ID (_id).This is useful for ensuring that the contract uses up-to-date oracle data for each asset, providing accurate pricing information.

   * require(_orb != address(0), "CurveLPOracle/invalid-orb");: This line checks whether the provided oracle address (_orb) is not zero (i.e., a valid address). If it is zero, it raises an error indicating an invalid oracle address.

   * require(_id < ncoins, "CurveLPOracle/invalid-orb-index");: This line checks whether the provided asset ID (_id) is within the valid range. The number of underlying assets in the Curve pool is stored in the ncoins state variable, and this check ensures that the provided ID is within a valid range.

   * orbs[_id] = _orb;: This line updates the address of the oracle for the specified asset. It assigns the provided oracle address (_orb) to the corresponding position in the orbs array based on the provided asset ID (_id).

  * emit Link(_id, _orb);: This line emits a Link event to log the linkage of the new oracle address (_orb) with the specified asset ID (_id). Events are used for logging important actions within a smart contract.

### function zzz() external view returns (uint256): 
        function zzz() external view returns (uint256) {
        if (zph == 0) return 0;  // backwards compatibility
        return zph - hop;
    }
    This function is publicly callable (external) and does not modify the contract's state. It returns a uint256 value.it is used to retrieve the time remaining until the next expected price update. It checks for backward compatibility and calculates this time difference based on the zph and hop values. This information can be useful for external systems or users to determine when to expect the next price update within the contract.

   * if (zph == 0) return 0;: This line checks if the zph state variable, which represents the time of the last price update plus the hop value, is equal to zero. If it is, this condition is used for backward compatibility, and the function returns 0. This is likely a special case to handle scenarios where there might not have been any price updates.

    * return zph - hop;: If the zph is not zero, this line calculates the difference between the zph value and the hop value. This calculation essentially provides the time remaining until the next expected price update. It subtracts the time that has already passed (represented by hop) from the total time interval between updates.

### function pass() external view returns (bool):
         function pass() external view returns (bool) {
        return block.timestamp >= zph;
    }
    This function is publicly callable (external) and does not modify the contract's state. It returns a boolean value (bool) that indicates whether enough time has passed since the last price update.its purpose is to provide a simple way to check whether sufficient time has passed since the last price update within the contract. It returns true when the time condition is met, allowing external systems or users to determine whether it's appropriate to trigger a new price update.

  * return block.timestamp >= zph;: This line checks whether the current block timestamp, obtained using block.timestamp, is        greater than or equal to the zph value. The zph value represents the time of the last price update plus the hop value, which is the expected time interval between price updates.

    * If block.timestamp is greater than or equal to zph, it means that enough time has passed, and the function returns true.
    If block.timestamp is less than zph, it indicates that the expected time for the next price update has not yet arrived, and the function returns false.

### function poke() external payable:
         function poke() external payable {

        // Ensure a single SLOAD while avoiding solc's excessive bitmasking bureaucracy.
        uint256 hop_;
        {

            // Block-scoping these variables saves some gas.
            uint256 stopped_;
            uint256 zph_;
            assembly {
                let slot1 := sload(1)
                stopped_  := and(slot1,         0xff  )
                hop_      := and(shr(8, slot1), 0xffff)
                zph_      := shr(24, slot1)
            }

            // When stopped, values are set to zero and should remain such; thus, disallow updating in that case.
            require(stopped_ == 0, "CurveLPOracle/is-stopped");

            // Equivalent to requiring that pass() returns true; logic repeated to save gas.
            require(block.timestamp >= zph_, "CurveLPOracle/not-passed");
        }

        uint256 val = type(uint256).max;
        for (uint256 i = 0; i < ncoins;) {
            uint256 price = OracleLike(orbs[i]).read();
            if (price < val) val = price;
            unchecked { i++; }
        }
        val = val * CurvePoolLike(pool).get_virtual_price() / 10**18;
        require(val != 0, "CurveLPOracle/zero-price");
        require(val <= type(uint128).max, "CurveLPOracle/price-overflow");
        Feed memory cur_ = nxt;
        cur = cur_;
        nxt = Feed(uint128(val), 1);

        assembly {
            sstore(
                1,
                add(
                    shl(24, add(timestamp(), hop_)),  // zph value starts 24 bits in
                    shl(8, hop_)                      // hop value starts 8 bits in
                )
            )
        }

        // This will revert if called during execution of a state-modifying pool function.
        if (nonreentrant) {
            uint256[2] calldata amounts;
            CurvePoolLike(pool).remove_liquidity(0, amounts);
        }

        emit Value(cur_.val, uint128(val));

        // Safe to terminate immediately since no postfix modifiers are applied.
        assembly { stop() }
    }
  * it's purpose is to update price information based on multiple oracles and performing various checks to ensure the validity and timeliness of the data. It also handles reentrancy concerns and logs price updates with events.
     
  * uint256 hop_;: This line declares a local variable hop_ of type uint256 to store the hop value temporarily.

  * Block-Scoped Variables: The code block enclosed in curly braces { ... } block-scopes several variables, which helps to save gas. These variables are stopped_, hop_, and zph_.

  * Assembly: The code uses assembly to efficiently load values from storage, particularly from a storage slot (slot1). It extracts the stopped_, hop_, and zph_ values from the storage slot and stores them in their respective local variables.

  * require(stopped_ == 0, "CurveLPOracle/is-stopped");: This line checks if the stopped_ value is equal to zero. If it is, it means that the contract is not stopped (meaning price updates are allowed). If stopped_ is not equal to zero, it raises an error, preventing price updates when the contract is stopped.

  * require(block.timestamp >= zph_, "CurveLPOracle/not-passed");: This line checks if the current block timestamp (block.timestamp) is greater than or equal to the zph_ value. This condition ensures that enough time has passed since the last price update. If the condition is not met, it raises an error, indicating that the update is not yet due.

  * Price Aggregation: The function then aggregates prices from various oracles for each of the underlying assets in the Curve pool. It finds the minimum price among them and stores it in the val variable.

  * Price Calculation: The code then calculates the final price (val) by multiplying the minimum price obtained in the previous step by the virtual price of the Curve pool. It divides this result by 10^18.

  ## Error Checks: There are two error checks:

  * require(val != 0, "CurveLPOracle/zero-price");: Checks that the calculated val is not zero, indicating a valid price.
    
  * require(val <= type(uint128).max, "CurveLPOracle/price-overflow");: Ensures that the calculated val does not overflow a uint128 data type.

  * Price Update: The function updates the current (cur) and queued (nxt) price feeds with the newly calculated price.

  * Storage Update: The assembly block efficiently updates the storage values related to zph and hop. It also accounts for the new time interval for the next update.

  * Reentrancy Check: If the nonreentrant flag is set to true, it calls a function in the Curve pool contract, remove_liquidity(0, amounts). This is a safety measure to prevent reentrancy attacks in situations where the Curve pool contract might invoke a callback function that could lead to reentrancy.

  * Event Emission: An emit Value(cur_.val, uint128(val)) line logs the change in price with a Value event. This event helps in tracking price updates.

  * Termination: The function terminates using assembly, specifically assembly { stop() }. This safely stops the function without applying any postfix modifiers.

### function peek() external view toll returns (bytes32, bool):
        function peek() external view toll returns (bytes32,bool) {
        return (bytes32(uint256(cur.val)), cur.has == 1);
    }
    This function is publicly callable (external) and does not modify the contract's state. it provides a way for external systems or users to query the current price and its validity for the tracked asset. It returns a tuple with the price value and a boolean flag to indicate whether the price is considered valid or not. This information can be useful for making informed decisions based on the asset's current price within the contract. The toll modifier ensures that only whitelisted addresses can access this information.

   * return (bytes32(uint256(cur.val)), cur.has == 1);: This line constructs the return values for the function:

   * bytes32(uint256(cur.val)): This part converts the cur.val (the current price) into a bytes32 type. It essentially casts the price to a bytes32 value.
    cur.has == 1: This part checks if the cur.has value is equal to 1. If it is, it returns true to indicate that the price is considered valid. If cur.has is not equal to 1, it returns false, indicating that the price may not be valid.

### function peep() external view toll returns (bytes32, bool):
    function peep() external view toll returns (bytes32,bool) {
        return (bytes32(uint256(nxt.val)), nxt.has == 1);
    }
    This function is publicly callable (external) and does not modify the contract's state. it provides a way for external systems or users to query the queued (next) price and its validity for the tracked asset. It returns a tuple with the queued price value and a boolean flag to indicate whether the queued price is considered valid or not. This information can be useful for making informed decisions based on the asset's next expected price within the contract. The toll modifier ensures that only whitelisted addresses can access this information.

   * return (bytes32(uint256(nxt.val)), nxt.has == 1);: This line constructs the return values for the function:

   * bytes32(uint256(nxt.val)): This part converts the nxt.val (the queued price) into a bytes32 type. It essentially casts the queued price to a bytes32 value.
    
   * nxt.has == 1: This part checks if the nxt.has value is equal to 1. If it is, it returns true to indicate that the queued price is considered valid. If nxt.has is not equal to 1, it returns false, indicating that the queued price may not be valid.

### function read() external view toll returns (bytes32):
         function read() external view toll returns (bytes32) {
        require(cur.has == 1, "CurveLPOracle/no-current-value");
        return (bytes32(uint256(cur.val)));
    }
    This function is publicly callable (external) and does not modify the contract's state. it provides a way for external systems or users to query the current price for the tracked asset within the contract. It checks for the validity of the current price and returns the price as a bytes32 value. The toll modifier ensures that only whitelisted addresses can access this information.

   * require(cur.has == 1, "CurveLPOracle/no-current-value");: This line checks if the cur.has value is equal to 1. The cur.has value is a flag indicating whether the current price is considered valid. If it is equal to 1, it means that the current price is considered valid. If cur.has is not equal to 1, it raises an error with the message "CurveLPOracle/no-current-value," indicating that there is no valid current price available.

   * return (bytes32(uint256(cur.val));: This line constructs the return value for the function. It converts the cur.val (the current price) into a bytes32 type by casting it to a bytes32 value.

### function kiss(address _a) external auth:
        function kiss(address _a) external auth {
        require(_a != address(0), "CurveLPOracle/no-contract-0");
        bud[_a] = 1;
        emit Kiss(_a);
    }
    This function is publicly callable (external) and requires that the caller has authorization (auth) to execute it. It allows authorized administrators to add a specific address to the whitelist maintained by the CurveLPOracle contract. Addresses on the whitelist are granted special privileges, such as access to certain functions or data. The function checks that a valid non-zero address is provided before adding it to the whitelist and logs the action with an event.

  * require(_a != address(0), "CurveLPOracle/no-contract-0");: This line checks that the provided address _a is not the zero address (0x0). Adding the zero address to the whitelist is not allowed, and this condition ensures that a valid address is provided.

  * bud[_a] = 1;: If the provided address _a is valid, this line sets the value of bud[_a] to 1, effectively adding the address to the whitelist.

  * emit Kiss(_a);: This line emits a Kiss event with the address _a as an argument. The event is logged to record the addition of the address to the whitelist.

### function kiss(address[] calldata _a) external auth:
        function kiss(address[] calldata _a) external auth {
        for(uint256 i = 0; i < _a.length;) {
            require(_a[i] != address(0), "CurveLPOracle/no-contract-0");
            bud[_a[i]] = 1;
            emit Kiss(_a[i]);
            unchecked { i++; }
        }
    }
    This function is publicly callable (external) and requires that the caller has authorization (auth) to execute it. the kiss(address[] calldata _a) function allows authorized administrators to add multiple addresses to the whitelist in a single transaction. It checks that each provided address is valid (non-zero), adds them to the whitelist, and logs each addition with a corresponding Kiss event. This is a convenient way to manage multiple addresses on the whitelist in a single operation.

    The function uses a for loop to iterate through the provided array of addresses.

   * for(uint256 i = 0; i < _a.length;) { ... }: This loop iterates through each address in the _a array.
    require(_a[i] != address(0), "CurveLPOracle/no-contract-0");: Within the loop, it checks that the current address _a[i] is not the zero address (0x0). Adding the zero address to the whitelist is not allowed, and this condition ensures that valid non-zero addresses are provided.
    If the address is valid (non-zero), the function does the following:

   * bud[_a[i]] = 1;: It sets the value of bud[_a[i]] to 1, effectively adding the current address to the whitelist.
    emit Kiss(_a[i]);: It emits a Kiss event with the current address _a[i] as an argument. This event is logged to record the addition of each address to the whitelist.
    The loop continues to process all addresses in the _a array, and once all valid addresses have been processed, the function returns.

### function diss(address[] calldata _a) external auth:
         function diss(address _a) external auth {
        bud[_a] = 0;
        emit Diss(_a);
    }
    function diss(address[] calldata _a) external auth {
        for(uint256 i = 0; i < _a.length;) {
            bud[_a[i]] = 0;
            emit Diss(_a[i]);
            unchecked { i++; }
        }
    }


    This function is publicly callable (external) and requires that the caller has authorization (auth) to execute it.  The diss(address[] calldata _a) function allows authorized administrators to remove multiple addresses from the whitelist in a single transaction. It iterates through the provided array of addresses, sets their whitelist status to 0 (removing them), and logs each removal with a corresponding Diss event. This is a convenient way to manage the whitelist by removing multiple addresses in a single operation.

    The function uses a for loop to iterate through the provided array of addresses.

   * for(uint256 i = 0; i < _a.length;) { ... }: This loop iterates through each address in the _a array.
    For each address in the loop, the function does the following:

   * bud[_a[i]] = 0;: It sets the value of bud[_a[i]] to 0, effectively removing the current address from the whitelist.
    emit Diss(_a[i]);: It emits a Diss event with the current address _a[i] as an argument. This event is logged to record the removal of each address from the whitelist.
    The loop continues to process all addresses in the _a array, and once all addresses have been processed, the function returns.

### receive() external payable {};:
    Needed for Curve pool reentrancy checks, since remove_liquidity with call the caller to do an ETH transfer.
    Technically we receive 0 Ether, but using a payable function saves a little gas.
    A fallback function marked as payable is included for potential reentrancy checks in the Curve pool.


### CONCLUSION;:
    For good practice i suggest variable name and function name should give an hint of what it is implementing, that was one of the major chanllenge i encountered when reviewing this code.

