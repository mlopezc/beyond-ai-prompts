# ADR-009: Polygon (Layer 2) como Blockchain Principal

## Estado
Aceptado

## Contexto

ArtChain Auction Platform necesita blockchain para:
- **Certificación de autenticidad:** Cada artículo debe tener certificado digital inmutable
- **Registro de pujas:** Transparencia y anti-fraude en proceso de bidding
- **NFT de propiedad:** Certificado de ownership transferible
- **Trazabilidad:** Historial completo de propiedad y transacciones

**Requisitos del PRD relacionados:**
- RF-006: Certificación de autenticidad (CRÍTICA)
- RF-007: Validación de pujas en blockchain (CRÍTICA)
- Confirmación de puja: <2 segundos
- 10,000 usuarios concurrentes (potencialmente miles de transacciones/día)
- Budget: $5K/mes en gas fees

**Características necesarias blockchain:**
- **Latencia baja:** Confirmación de transacciones en segundos (no minutos)
- **Costo bajo:** Transacciones económicas para viabilidad del negocio
- **Seguridad:** Network segura y probada
- **Compatibilidad:** EVM-compatible (Solidity, herramientas existentes)
- **Escalabilidad:** Soportar alta frecuencia de transacciones
- **Finality:** Confirmaciones rápidas e irreversibles

**Restricciones:**
- Timeline: 4 meses MVP (no hay tiempo para blockchain custom)
- Equipo: 1 blockchain engineer
- Budget gas: <$0.10 por transacción idealmente

## Decisión

Adoptaremos **Polygon PoS (Proof of Stake)** como blockchain principal, con **Ethereum Mainnet** como fallback para artículos de ultra alto valor (>$1M).

### Por qué Polygon

#### 1. Performance y Costo

**Polygon:**
- ⚡ **Velocidad:** 2 segundos por bloque (vs 12-15s Ethereum)
- 💰 **Costo:** ~$0.01-0.05 por transacción (vs $5-50+ Ethereum)
- 🚀 **Throughput:** 7,000+ TPS (vs 15-30 TPS Ethereum)
- ✅ **Finality:** <30 segundos (vs 6-12 min Ethereum)

**Cálculo de costos:**
```
Escenario conservador:
- 1,000 subastas/mes
- 50 pujas promedio por subasta
- 1,000 certificaciones de autenticidad/mes

Transacciones totales = 1,000 + (1,000 * 50) + 1,000 = 52,000 tx/mes

Polygon: 52,000 * $0.02 = $1,040/mes ✅
Ethereum: 52,000 * $10 = $520,000/mes ❌ (inviable)
```

#### 2. EVM Compatibility

- **Solidity:** Mismo lenguaje que Ethereum
- **Tooling:** Hardhat, Truffle, Remix funcionan sin cambios
- **Libraries:** Web3.js, Ethers.js compatibles
- **Migration path:** Fácil migrar a Ethereum si necesario

#### 3. Seguridad y Madurez

- **Validators:** 100+ validadores
- **TVL:** $1B+ Total Value Locked
- **Auditorías:** Multiple security audits
- **Track record:** 3+ años en producción sin hacks mayores
- **Bridges:** Puentes seguros a Ethereum

#### 4. Ecosystem

- **NFT Standards:** ERC-721, ERC-1155 soportados
- **Marketplaces:** OpenSea, Rarible integrados
- **Wallets:** MetaMask, Coinbase Wallet, etc.
- **Infrastructure:** Alchemy, Infura, QuickNode soportan Polygon

### Arquitectura de Integración

#### Smart Contracts

**1. ItemAuthenticity.sol** - Certificación de autenticidad
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/access/AccessControl.sol";
import "@openzeppelin/contracts/token/ERC721/ERC721.sol";

contract ItemAuthenticity is ERC721, AccessControl {
    bytes32 public constant CERTIFIER_ROLE = keccak256("CERTIFIER_ROLE");

    struct Certificate {
        string itemId;           // ArtChain internal ID
        string metadataURI;      // IPFS hash with full metadata
        string artist;
        uint256 year;
        uint256 certifiedAt;
        address certifier;
        bool exists;
    }

    mapping(string => Certificate) public certificates;
    mapping(uint256 => string) public tokenIdToItemId;

    uint256 private _tokenIdCounter;

    event ItemCertified(
        string indexed itemId,
        uint256 indexed tokenId,
        address certifier,
        string metadataURI
    );

    constructor() ERC721("ArtChain Certificate", "ARTC") {
        _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);
        _grantRole(CERTIFIER_ROLE, msg.sender);
    }

    function certifyItem(
        string memory itemId,
        string memory metadataURI,
        string memory artist,
        uint256 year,
        address owner
    ) public onlyRole(CERTIFIER_ROLE) returns (uint256) {
        require(!certificates[itemId].exists, "Item already certified");

        uint256 tokenId = _tokenIdCounter++;

        certificates[itemId] = Certificate({
            itemId: itemId,
            metadataURI: metadataURI,
            artist: artist,
            year: year,
            certifiedAt: block.timestamp,
            certifier: msg.sender,
            exists: true
        });

        tokenIdToItemId[tokenId] = itemId;

        _safeMint(owner, tokenId);

        emit ItemCertified(itemId, tokenId, msg.sender, metadataURI);

        return tokenId;
    }

    function getCertificate(string memory itemId)
        public
        view
        returns (Certificate memory)
    {
        require(certificates[itemId].exists, "Certificate not found");
        return certificates[itemId];
    }

    function verifyCertificate(string memory itemId)
        public
        view
        returns (bool)
    {
        return certificates[itemId].exists;
    }

    function supportsInterface(bytes4 interfaceId)
        public
        view
        override(ERC721, AccessControl)
        returns (bool)
    {
        return super.supportsInterface(interfaceId);
    }
}
```

**2. AuctionBid.sol** - Registro de pujas
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract AuctionBid {
    struct Bid {
        string auctionId;
        address bidder;
        uint256 amount;
        uint256 timestamp;
        bytes32 offChainTxHash; // Hash de transacción en backend
    }

    mapping(string => Bid[]) public auctionBids;

    event BidPlaced(
        string indexed auctionId,
        address indexed bidder,
        uint256 amount,
        uint256 timestamp,
        bytes32 offChainTxHash
    );

    function recordBid(
        string memory auctionId,
        address bidder,
        uint256 amount,
        bytes32 offChainTxHash
    ) public {
        Bid memory newBid = Bid({
            auctionId: auctionId,
            bidder: bidder,
            amount: amount,
            timestamp: block.timestamp,
            offChainTxHash: offChainTxHash
        });

        auctionBids[auctionId].push(newBid);

        emit BidPlaced(auctionId, bidder, amount, block.timestamp, offChainTxHash);
    }

    function getBids(string memory auctionId)
        public
        view
        returns (Bid[] memory)
    {
        return auctionBids[auctionId];
    }

    function getBidCount(string memory auctionId)
        public
        view
        returns (uint256)
    {
        return auctionBids[auctionId].length;
    }
}
```

#### Backend Integration (Node.js Blockchain Service)

```typescript
// src/services/blockchain.service.ts
import { ethers } from 'ethers';
import { create } from 'ipfs-http-client';

export class BlockchainService {
  private provider: ethers.providers.JsonRpcProvider;
  private wallet: ethers.Wallet;
  private authenticityContract: ethers.Contract;
  private bidContract: ethers.Contract;
  private ipfs: any;

  constructor() {
    // Polygon Mumbai (testnet) or Mainnet
    this.provider = new ethers.providers.JsonRpcProvider(
      process.env.POLYGON_RPC_URL || 'https://polygon-rpc.com'
    );

    this.wallet = new ethers.Wallet(
      process.env.BLOCKCHAIN_PRIVATE_KEY!,
      this.provider
    );

    this.authenticityContract = new ethers.Contract(
      process.env.AUTHENTICITY_CONTRACT_ADDRESS!,
      ItemAuthenticityABI,
      this.wallet
    );

    this.bidContract = new ethers.Contract(
      process.env.BID_CONTRACT_ADDRESS!,
      AuctionBidABI,
      this.wallet
    );

    // IPFS via Pinata
    this.ipfs = create({
      host: 'ipfs.infura.io',
      port: 5001,
      protocol: 'https',
      headers: {
        authorization: `Basic ${Buffer.from(
          `${process.env.INFURA_PROJECT_ID}:${process.env.INFURA_SECRET}`
        ).toString('base64')}`,
      },
    });
  }

  async certifyItem(item: {
    id: string;
    title: string;
    artist: string;
    year: number;
    images: string[];
    description: string;
    ownerId: string;
  }): Promise<{ txHash: string; tokenId: number; ipfsHash: string }> {
    try {
      // 1. Upload metadata to IPFS
      const metadata = {
        name: item.title,
        artist: item.artist,
        year: item.year,
        description: item.description,
        images: item.images,
        certifiedAt: new Date().toISOString(),
      };

      const { cid } = await this.ipfs.add(JSON.stringify(metadata));
      const ipfsHash = `ipfs://${cid}`;

      // 2. Get owner's blockchain address (from user profile)
      const ownerAddress = await this.getUserBlockchainAddress(item.ownerId);

      // 3. Call smart contract
      const tx = await this.authenticityContract.certifyItem(
        item.id,
        ipfsHash,
        item.artist,
        item.year,
        ownerAddress,
        {
          gasLimit: 300000, // Estimate gas
        }
      );

      // 4. Wait for confirmation
      const receipt = await tx.wait(1); // Wait for 1 confirmation

      // 5. Extract tokenId from event
      const event = receipt.events?.find((e: any) => e.event === 'ItemCertified');
      const tokenId = event?.args?.tokenId.toNumber();

      return {
        txHash: receipt.transactionHash,
        tokenId,
        ipfsHash,
      };
    } catch (error) {
      console.error('Certification failed:', error);
      throw new Error(`Failed to certify item: ${error.message}`);
    }
  }

  async recordBid(bid: {
    auctionId: string;
    userId: string;
    amount: number;
    internalTxId: string;
  }): Promise<{ txHash: string }> {
    try {
      const bidderAddress = await this.getUserBlockchainAddress(bid.userId);

      // Hash of internal transaction (for linking)
      const offChainTxHash = ethers.utils.keccak256(
        ethers.utils.toUtf8Bytes(bid.internalTxId)
      );

      const tx = await this.bidContract.recordBid(
        bid.auctionId,
        bidderAddress,
        ethers.utils.parseEther(bid.amount.toString()),
        offChainTxHash,
        {
          gasLimit: 150000,
        }
      );

      const receipt = await tx.wait(1);

      return { txHash: receipt.transactionHash };
    } catch (error) {
      console.error('Bid recording failed:', error);
      throw new Error(`Failed to record bid: ${error.message}`);
    }
  }

  async verifyCertificate(itemId: string): Promise<{
    exists: boolean;
    metadata?: any;
  }> {
    try {
      const certificate = await this.authenticityContract.getCertificate(itemId);

      if (!certificate.exists) {
        return { exists: false };
      }

      // Fetch metadata from IPFS
      const ipfsHash = certificate.metadataURI.replace('ipfs://', '');
      const metadata = await this.fetchFromIPFS(ipfsHash);

      return {
        exists: true,
        metadata: {
          ...certificate,
          ...metadata,
        },
      };
    } catch (error) {
      return { exists: false };
    }
  }

  private async fetchFromIPFS(cid: string): Promise<any> {
    const chunks = [];
    for await (const chunk of this.ipfs.cat(cid)) {
      chunks.push(chunk);
    }
    return JSON.parse(Buffer.concat(chunks).toString());
  }

  private async getUserBlockchainAddress(userId: string): Promise<string> {
    // Fetch from user service
    // For MVP, could be stored in user profile or generated deterministically
    // from userId (HD wallet derivation)
    return '0x...'; // Placeholder
  }

  async getGasPrice(): Promise<{ fast: string; average: string }> {
    const feeData = await this.provider.getFeeData();
    return {
      fast: ethers.utils.formatUnits(feeData.maxFeePerGas || 0, 'gwei'),
      average: ethers.utils.formatUnits(feeData.gasPrice || 0, 'gwei'),
    };
  }
}
```

### Deployment Strategy

**Fase 1 (MVP):**
- Deploy en Polygon Mumbai Testnet
- Testing completo sin costos reales

**Fase 2 (Production):**
- Deploy en Polygon Mainnet
- Monitoreo de gas costs
- Subsidio de gas para primeros usuarios

**Ethereum Mainnet:**
- Solo para artículos >$1M (extra security)
- Usuario paga gas fees
- Mayor tiempo de confirmación aceptable

### Fallback y Contingency

**Si Polygon tiene issues:**
1. **BNB Smart Chain (BSC):** EVM-compatible, bajo costo
2. **Arbitrum:** Layer 2 de Ethereum, similar performance
3. **Optimism:** Layer 2, más descentralizado que BSC

**Abstracción:**
```typescript
interface BlockchainProvider {
  certifyItem(...): Promise<...>;
  recordBid(...): Promise<...>;
  verifyCertificate(...): Promise<...>;
}

class PolygonProvider implements BlockchainProvider { ... }
class EthereumProvider implements BlockchainProvider { ... }

// Factory pattern
const blockchain = BlockchainFactory.create(config.network);
```

## Consecuencias

### Positivas

1. **Bajo costo operacional**
   - $0.01-0.05 por tx vs $5-50 en Ethereum
   - Budget de $5K/mes cubre ~100K-500K transacciones

2. **Latencia baja**
   - 2 segundos por bloque
   - Confirmaciones en <30 segundos
   - Cumple requisito de <2s para pujas (optimistic update + background confirm)

3. **Escalabilidad**
   - 7,000 TPS soporta 10K usuarios concurrentes
   - Sin preocupación de congestión

4. **EVM Compatibility**
   - Reutilizar código Solidity existente
   - Tooling maduro (Hardhat, Remix)
   - Fácil migración a Ethereum si necesario

5. **Ecosystem**
   - Wallets: MetaMask, Trust Wallet, etc.
   - Marketplaces: OpenSea soporta Polygon
   - Infrastructure: Alchemy, Infura, QuickNode

6. **Seguridad probada**
   - 3+ años en producción
   - $1B+ TVL
   - Múltiples auditorías

7. **Developer Experience**
   - Web3.js, Ethers.js funcionan sin cambios
   - Testing con Hardhat
   - Faucets para testnet

### Negativas

1. **Menos descentralizado que Ethereum**
   - ~100 validadores vs miles en Ethereum
   - Centralización relativa (mitigado por PoS)

2. **Bridge risk**
   - Puente Polygon-Ethereum es punto de fallo
   - Historial de hacks en bridges (aunque Polygon oficial es seguro)

3. **Dependencia en Polygon**
   - Si Polygon falla o cierra, necesitamos migrar
   - Mitigado con abstracción de código

4. **Percepción**
   - Algunos usuarios pueden preferir Ethereum mainnet
   - "No es el blockchain real" (aunque técnicamente Layer 2)

5. **Finality no absoluta**
   - Confirmaciones pueden revertirse teóricamente
   - Aunque probabilidad muy baja después de 30+ segundos

### Neutrales

1. **Multi-chain future**
   - Podríamos expandir a múltiples chains
   - Polygon + Ethereum + Solana, etc.

2. **Gas fee volatility**
   - Aunque bajo, puede fluctuar
   - Monitoring necesario

## Alternativas Consideradas

### Alternativa 1: Ethereum Mainnet
**Ventajas:**
- Más descentralizado y seguro
- Network effect más fuerte
- Percepción de "real blockchain"

**Razones para rechazar:**
- **Costo:** $5-50 por tx (inviable para modelo de negocio)
- **Latencia:** 12-15s por bloque, 6-12 min finality
- **Escalabilidad:** 15-30 TPS (insuficiente)
- **Decisión:** Solo para ultra alto valor (>$1M)

### Alternativa 2: Solana
**Ventajas:**
- Muy rápido (400ms blocks)
- Muy bajo costo (<$0.001 por tx)
- Alto throughput (50K TPS)

**Razones para rechazar:**
- **NO EVM-compatible:** Diferente lenguaje (Rust, no Solidity)
- **Outages:** Historial de network downtime
- **Ecosystem:** Menos maduro para NFTs (comparado con Ethereum ecosystem)
- **Learning curve:** Equipo tendría que aprender Rust
- **Timeline:** 4 meses no permite

### Alternativa 3: Avalanche
**Ventajas:**
- EVM-compatible
- Muy rápido (1-2s finality)
- Bajo costo (~$0.05 por tx)

**Razones para rechazar:**
- **Ecosystem:** Menor que Polygon
- **Adopción:** Menos wallets y marketplaces
- **Team expertise:** Equipo tiene más experiencia con Ethereum/Polygon

### Alternativa 4: BNB Smart Chain (BSC)
**Ventajas:**
- EVM-compatible
- Bajo costo (~$0.10 per tx)
- Rápido (3s blocks)

**Razones para rechazar:**
- **Centralización:** Muy centralizado (21 validadores)
- **Percepción:** "Not a real blockchain"
- **Security concerns:** Menos auditorías que Polygon

### Alternativa 5: Flow (Dapper Labs)
**Ventajas:**
- Diseñado para NFTs
- Usado por NBA Top Shot

**Razones para rechazar:**
- **NO EVM-compatible:** Lenguaje Cadence
- **Learning curve:** Nueva tecnología
- **Ecosystem:** Menor para NFTs generales
- **Timeline:** No factible en 4 meses

### Alternativa 6: Private Blockchain (Hyperledger)
**Ventajas:**
- Control total
- Sin gas fees
- Privacy

**Razones para rechazar:**
- **NO transparencia pública:** Contradice propuesta de valor
- **Overhead:** Mantenimiento de network
- **Percepción:** "No es blockchain real"

## Referencias

- PRD RF-006, RF-007
- Polygon Documentation: https://docs.polygon.technology/
- Ethereum vs Layer 2 Comparison: https://l2beat.com/
- Hardhat: https://hardhat.org/

## Notas

### Gas Optimization

**Batch transactions:**
```solidity
function batchCertify(
    string[] memory itemIds,
    string[] memory metadataURIs,
    ...
) public onlyRole(CERTIFIER_ROLE) {
    for (uint i = 0; i < itemIds.length; i++) {
        _certifyItem(itemIds[i], metadataURIs[i], ...);
    }
}
```

**Use events vs storage:**
- Events más baratos que storage
- Para datos históricos, usar events y indexar off-chain

**Struct packing:**
```solidity
// Bien (packed en un slot)
struct Certificate {
    uint128 year;
    uint128 certifiedAt;
    address certifier; // 20 bytes
}

// Mal (múltiples slots)
struct Certificate {
    uint256 year;
    address certifier;
    uint256 certifiedAt;
}
```

### Monitoring

**Metrics to track:**
- Gas price (alert si >$0.10)
- Transaction success rate
- Confirmation time (p95, p99)
- Contract balance (para subsidio de gas)

## Historial

- 2025-11-07: Decisión inicial (Aceptado)
