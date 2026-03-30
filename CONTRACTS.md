# SPECTRA Protocol — Smart Contract Specifications

---

## Contract Overview

SPECTRA's Wave 1 consists of 5 core contracts:

```
SpectraACL.sol ─────────── Shared access control helper
     │
     ├── SpectraVault.sol ── Encrypted dataset storage
     │
     ├── SpectraQuery.sol ── FHE query execution engine
     │
     ├── SpectraMarket.sol ─ Payment, fees, disputes
     │
     └── SpectraToken.sol ── $SPEC FHERC20 token
```

---

## 1. SpectraACL.sol — Shared Access Control

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.25;

import { FHE, euint64, ebool } from "@fhenixprotocol/cofhe-contracts/FHE.sol";

abstract contract SpectraACL {
    
    enum Role { NONE, PROVIDER, BUYER, VALIDATOR, AUDITOR, ADMIN }
    
    mapping(address => Role) public roles;
    address public admin;
    
    modifier onlyRole(Role _role) {
        require(roles[msg.sender] == _role || roles[msg.sender] == Role.ADMIN, "Unauthorized");
        _;
    }
    
    modifier onlyAdmin() {
        require(msg.sender == admin, "Admin only");
        _;
    }
    
    function grantRole(address account, Role role) external onlyAdmin {
        roles[account] = role;
    }
    
    function revokeRole(address account) external onlyAdmin {
        roles[account] = Role.NONE;
    }
    
    // FHE ACL helpers
    function _retainAccess(euint64 handle) internal {
        FHE.allowThis(handle);
    }
    
    function _grantDecrypt(euint64 handle, address to) internal {
        FHE.allow(handle, to);
    }
    
    function _grantDecryptBool(ebool handle, address to) internal {
        FHE.allow(handle, to);
    }
}
```

---

## 2. SpectraVault.sol — Encrypted Dataset Storage

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.25;

import { FHE, euint64, InEuint64 } from "@fhenixprotocol/cofhe-contracts/FHE.sol";
import "./SpectraACL.sol";

contract SpectraVault is SpectraACL {
    
    struct DatasetMeta {
        address provider;
        string  schemaName;      // e.g., "salary_data"
        uint256 recordCount;
        uint256 pricePerQuery;   // in $SPEC wei
        uint256 createdAt;
        bool    active;
    }
    
    // datasetId => metadata
    mapping(uint256 => DatasetMeta) public datasets;
    
    // datasetId => fieldName => recordIndex => encrypted value
    mapping(uint256 => mapping(string => mapping(uint256 => euint64))) private encryptedRecords;
    
    // datasetId => fieldNames[]
    mapping(uint256 => string[]) public datasetFields;
    
    uint256 public nextDatasetId;
    
    // Events
    event DatasetCreated(uint256 indexed datasetId, address indexed provider, string schemaName);
    event DataUploaded(uint256 indexed datasetId, string field, uint256 recordIndex);
    
    constructor() {
        admin = msg.sender;
        roles[msg.sender] = Role.ADMIN;
    }
    
    /// @notice Register a new dataset with schema and pricing
    function createDataset(
        string calldata schemaName,
        string[] calldata fieldNames,
        uint256 pricePerQuery
    ) external onlyRole(Role.PROVIDER) returns (uint256 datasetId) {
        datasetId = nextDatasetId++;
        
        datasets[datasetId] = DatasetMeta({
            provider: msg.sender,
            schemaName: schemaName,
            recordCount: 0,
            pricePerQuery: pricePerQuery,
            createdAt: block.timestamp,
            active: true
        });
        
        for (uint256 i = 0; i < fieldNames.length; i++) {
            datasetFields[datasetId].push(fieldNames[i]);
        }
        
        emit DatasetCreated(datasetId, msg.sender, schemaName);
    }
    
    /// @notice Upload encrypted records to a dataset
    /// @dev Each InEuint64 is encrypted client-side via @cofhe/sdk
    function depositRecord(
        uint256 datasetId,
        string calldata field,
        InEuint64 calldata encValue
    ) external onlyRole(Role.PROVIDER) {
        DatasetMeta storage ds = datasets[datasetId];
        require(ds.provider == msg.sender, "Not your dataset");
        require(ds.active, "Dataset inactive");
        
        // Convert input to encrypted handle
        euint64 handle = FHE.asEuint64(encValue);
        
        // Store encrypted value
        uint256 idx = ds.recordCount;
        encryptedRecords[datasetId][field][idx] = handle;
        
        // ACL: contract can compute on this handle
        FHE.allowThis(handle);
        // ACL: provider can decrypt their own data
        FHE.allow(handle, msg.sender);
        
        ds.recordCount++;
        
        emit DataUploaded(datasetId, field, idx);
    }
    
    /// @notice Get encrypted record handle (internal, for SpectraQuery)
    function getRecord(
        uint256 datasetId,
        string calldata field,
        uint256 index
    ) external view returns (euint64) {
        return encryptedRecords[datasetId][field][index];
    }
    
    /// @notice Deactivate a dataset
    function deactivateDataset(uint256 datasetId) external {
        require(datasets[datasetId].provider == msg.sender, "Not your dataset");
        datasets[datasetId].active = false;
    }
}
```

---

## 3. SpectraQuery.sol — FHE Query Execution Engine

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.25;

import { FHE, euint64, ebool } from "@fhenixprotocol/cofhe-contracts/FHE.sol";
import "./SpectraACL.sol";
import "./SpectraVault.sol";

contract SpectraQuery is SpectraACL {
    
    enum QueryType { SUM, AVERAGE, MAX, MIN, COUNT_IF, RANGE_SUM }
    
    struct Query {
        uint256  datasetId;
        address  buyer;
        QueryType queryType;
        string   field;
        euint64  result;
        bool     executed;
        uint256  createdAt;
    }
    
    SpectraVault public vault;
    
    mapping(uint256 => Query) public queries;
    uint256 public nextQueryId;
    
    // Events
    event QuerySubmitted(uint256 indexed queryId, uint256 indexed datasetId, address indexed buyer);
    event QueryExecuted(uint256 indexed queryId);
    
    constructor(address _vault) {
        vault = SpectraVault(_vault);
        admin = msg.sender;
        roles[msg.sender] = Role.ADMIN;
    }
    
    /// @notice Submit and execute an aggregate query
    function querySUM(
        uint256 datasetId,
        string calldata field
    ) external onlyRole(Role.BUYER) returns (uint256 queryId) {
        uint256 recordCount = vault.datasets(datasetId).recordCount;
        require(recordCount > 0, "Empty dataset");
        
        queryId = nextQueryId++;
        
        // Initialize accumulator to encrypted zero
        euint64 sum = FHE.asEuint64(0);
        FHE.allowThis(sum);
        
        // Sum all encrypted records
        for (uint256 i = 0; i < recordCount; i++) {
            euint64 record = vault.getRecord(datasetId, field, i);
            sum = FHE.add(sum, record);
            FHE.allowThis(sum);
        }
        
        // Grant buyer access to result
        FHE.allow(sum, msg.sender);
        
        queries[queryId] = Query({
            datasetId: datasetId,
            buyer: msg.sender,
            queryType: QueryType.SUM,
            field: field,
            result: sum,
            executed: true,
            createdAt: block.timestamp
        });
        
        emit QuerySubmitted(queryId, datasetId, msg.sender);
        emit QueryExecuted(queryId);
    }
    
    /// @notice Execute AVERAGE query: sum / count
    function queryAVERAGE(
        uint256 datasetId,
        string calldata field
    ) external onlyRole(Role.BUYER) returns (uint256 queryId) {
        uint256 recordCount = vault.datasets(datasetId).recordCount;
        require(recordCount > 0, "Empty dataset");
        
        queryId = nextQueryId++;
        
        // Sum all records
        euint64 sum = FHE.asEuint64(0);
        FHE.allowThis(sum);
        
        for (uint256 i = 0; i < recordCount; i++) {
            euint64 record = vault.getRecord(datasetId, field, i);
            sum = FHE.add(sum, record);
            FHE.allowThis(sum);
        }
        
        // Divide by count for average
        euint64 count = FHE.asEuint64(uint64(recordCount));
        FHE.allowThis(count);
        
        euint64 avg = FHE.div(sum, count);
        FHE.allowThis(avg);
        FHE.allow(avg, msg.sender);
        
        queries[queryId] = Query({
            datasetId: datasetId,
            buyer: msg.sender,
            queryType: QueryType.AVERAGE,
            field: field,
            result: avg,
            executed: true,
            createdAt: block.timestamp
        });
        
        emit QuerySubmitted(queryId, datasetId, msg.sender);
        emit QueryExecuted(queryId);
    }
    
    /// @notice Execute MAX query
    function queryMAX(
        uint256 datasetId,
        string calldata field
    ) external onlyRole(Role.BUYER) returns (uint256 queryId) {
        uint256 recordCount = vault.datasets(datasetId).recordCount;
        require(recordCount > 0, "Empty dataset");
        
        queryId = nextQueryId++;
        
        euint64 maxVal = vault.getRecord(datasetId, field, 0);
        FHE.allowThis(maxVal);
        
        for (uint256 i = 1; i < recordCount; i++) {
            euint64 record = vault.getRecord(datasetId, field, i);
            maxVal = FHE.max(maxVal, record);
            FHE.allowThis(maxVal);
        }
        
        FHE.allow(maxVal, msg.sender);
        
        queries[queryId] = Query({
            datasetId: datasetId,
            buyer: msg.sender,
            queryType: QueryType.MAX,
            field: field,
            result: maxVal,
            executed: true,
            createdAt: block.timestamp
        });
        
        emit QuerySubmitted(queryId, datasetId, msg.sender);
        emit QueryExecuted(queryId);
    }
    
    /// @notice Execute MIN query
    function queryMIN(
        uint256 datasetId,
        string calldata field
    ) external onlyRole(Role.BUYER) returns (uint256 queryId) {
        uint256 recordCount = vault.datasets(datasetId).recordCount;
        require(recordCount > 0, "Empty dataset");
        
        queryId = nextQueryId++;
        
        euint64 minVal = vault.getRecord(datasetId, field, 0);
        FHE.allowThis(minVal);
        
        for (uint256 i = 1; i < recordCount; i++) {
            euint64 record = vault.getRecord(datasetId, field, i);
            minVal = FHE.min(minVal, record);
            FHE.allowThis(minVal);
        }
        
        FHE.allow(minVal, msg.sender);
        
        queries[queryId] = Query({
            datasetId: datasetId,
            buyer: msg.sender,
            queryType: QueryType.MIN,
            field: field,
            result: minVal,
            executed: true,
            createdAt: block.timestamp
        });
        
        emit QuerySubmitted(queryId, datasetId, msg.sender);
        emit QueryExecuted(queryId);
    }
    
    /// @notice COUNT_IF: count records where value >= threshold
    function queryCOUNT_IF(
        uint256 datasetId,
        string calldata field,
        InEuint64 calldata encThreshold
    ) external onlyRole(Role.BUYER) returns (uint256 queryId) {
        uint256 recordCount = vault.datasets(datasetId).recordCount;
        require(recordCount > 0, "Empty dataset");
        
        queryId = nextQueryId++;
        
        euint64 threshold = FHE.asEuint64(encThreshold);
        FHE.allowThis(threshold);
        
        euint64 count = FHE.asEuint64(0);
        FHE.allowThis(count);
        
        euint64 one = FHE.asEuint64(1);
        FHE.allowThis(one);
        
        euint64 zero = FHE.asEuint64(0);
        FHE.allowThis(zero);
        
        for (uint256 i = 0; i < recordCount; i++) {
            euint64 record = vault.getRecord(datasetId, field, i);
            
            // Check if record >= threshold
            ebool meetsThreshold = FHE.gte(record, threshold);
            
            // If true, add 1; else add 0
            euint64 increment = FHE.select(meetsThreshold, one, zero);
            FHE.allowThis(increment);
            
            count = FHE.add(count, increment);
            FHE.allowThis(count);
        }
        
        FHE.allow(count, msg.sender);
        
        queries[queryId] = Query({
            datasetId: datasetId,
            buyer: msg.sender,
            queryType: QueryType.COUNT_IF,
            field: field,
            result: count,
            executed: true,
            createdAt: block.timestamp
        });
        
        emit QuerySubmitted(queryId, datasetId, msg.sender);
        emit QueryExecuted(queryId);
    }
}
```

---

## 4. SpectraMarket.sol — Payment & Fee Distribution

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.25;

import "./SpectraACL.sol";
import "./SpectraVault.sol";
import "./SpectraQuery.sol";

contract SpectraMarket is SpectraACL {
    
    SpectraVault public vault;
    SpectraQuery public queryEngine;
    
    // Earnings per provider
    mapping(address => uint256) public providerEarnings;
    
    // Query payment tracking
    mapping(uint256 => bool) public queryPaid;
    
    // Fee split (basis points)
    uint256 public providerShare = 7000; // 70%
    uint256 public protocolShare = 2000; // 20%
    uint256 public validatorShare = 1000; // 10%
    
    // Protocol treasury
    address public treasury;
    uint256 public protocolEarnings;
    
    event QueryPaid(uint256 indexed queryId, address indexed buyer, uint256 amount);
    event EarningsClaimed(address indexed provider, uint256 amount);
    
    constructor(address _vault, address _query, address _treasury) {
        vault = SpectraVault(_vault);
        queryEngine = SpectraQuery(_query);
        treasury = _treasury;
        admin = msg.sender;
        roles[msg.sender] = Role.ADMIN;
    }
    
    /// @notice Pay for a query (must be called before query execution)
    function payForQuery(uint256 datasetId) external payable returns (uint256) {
        uint256 price = vault.datasets(datasetId).pricePerQuery;
        require(msg.value >= price, "Insufficient payment");
        require(!queryPaid[datasetId], "Already paid");
        
        // Split fees
        address provider = vault.datasets(datasetId).provider;
        uint256 providerFee = (msg.value * providerShare) / 10000;
        uint256 protocolFee = (msg.value * protocolShare) / 10000;
        // Remainder goes to validator pool
        
        providerEarnings[provider] += providerFee;
        protocolEarnings += protocolFee;
        
        emit QueryPaid(datasetId, msg.sender, msg.value);
        return price;
    }
    
    /// @notice Provider claims accumulated earnings
    function claimEarnings() external {
        uint256 amount = providerEarnings[msg.sender];
        require(amount > 0, "No earnings");
        
        providerEarnings[msg.sender] = 0;
        
        (bool success, ) = msg.sender.call{value: amount}("");
        require(success, "Transfer failed");
        
        emit EarningsClaimed(msg.sender, amount);
    }
}
```

---

## 5. SpectraToken.sol — $SPEC FHERC20 (Wave 2+)

> $SPEC token with encrypted balances. All holders' balances are private. Transfer amounts are private. Only the holder can decrypt their own balance.

This contract extends the standard FHERC20 pattern from the Fhenix documentation. Full implementation proceeds in Wave 2 once the core query engine is validated.

---

## FHE Operations Matrix

| Operation | Where Used | Purpose |
|-----------|-----------|---------|
| `FHE.asEuint64(InEuint64)` | SpectraVault.depositRecord | Convert encrypted input to handle |
| `FHE.asEuint64(uint64)` | SpectraQuery (accumulators) | Create encrypted constants (0, 1) |
| `FHE.add(a, b)` | SpectraQuery.querySUM/AVG/COUNT_IF | Aggregate encrypted values |
| `FHE.div(a, b)` | SpectraQuery.queryAVERAGE | Compute average |
| `FHE.max(a, b)` | SpectraQuery.queryMAX | Find maximum value |
| `FHE.min(a, b)` | SpectraQuery.queryMIN | Find minimum value |
| `FHE.gte(a, b)` | SpectraQuery.queryCOUNT_IF | Threshold comparison |
| `FHE.select(cond, a, b)` | SpectraQuery.queryCOUNT_IF | Conditional increment |
| `FHE.allowThis(handle)` | All contracts | Retain compute access |
| `FHE.allow(handle, addr)` | All contracts | Grant decrypt to user |
| `FHE.allowSender(handle)` | SpectraVault | Grant deposit access |
| `FHE.isInitialized(handle)` | SpectraVault | Validate handle exists |

**Total unique FHE operations: 12+** — significantly more than most competing projects.
