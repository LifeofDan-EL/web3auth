# Web3Auth + Smart Contract Interaction Example 

This project demonstrates how to integrate Apple Login with [Web3Auth Nomodal](https://web3auth.io/docs/sdk/pnp/web/no-modal) in a React app for authenticating users and interacting with a Base Chain smart contract. 

The example covers login/logout functionality, deploying, reading from, and writing to a smart contract, and checking wallet balances. 

Here is a [Loom recording](https://www.loom.com/share/e2cdb5a77cdb4a67b53df7e0cfff230f?sid=e85fa7a7-cd18-47f9-9541-45b5e874c136) of how the app works. 


## Prerequisites 
- Node.js and npm installed 
- Basic knowledge of React 
- Base wallet with Base Sepolia ETH (for deploying and interacting with smart contracts on a test network) 
- An account on the Web3Auth [dev dashboard](https://dashboard.web3auth.io/).
- An account on the Auth0 [dev dashboard](https://manage.auth0.com/).

Note: When the Web3Auth is initiated and ready, an EOA contract address is created for you. You will have to send Base Sepolia ETH to the wallet address to interact with Base Chain Testnet\. 

Deploy a contract first before. Also take a look at the console logs to see the logs around the contract, which you can also confirm with the [Base Sepolia blockchain scanner](https://base-sepolia.blockscout.com/). 
## Building the react app

### Create react app and start server
```
npx create-react-app web3auth-base-chain
cd web3auth-base-chain
```

### Install dependencies
```javascript
npm install ethers @web3auth/no-modal @web3auth/base @web3auth/ethereum-provider @web3auth/openlogin-adapter 
```

### Imports and configuarions
```javascript
import React, { useState, useEffect } from 'react';
import { ethers } from 'ethers';
import { WALLET_ADAPTERS } from "@web3auth/base";
import { Web3AuthNoModal } from "@web3auth/no-modal";
import { OpenloginAdapter } from "@web3auth/openlogin-adapter";
import { EthereumPrivateKeyProvider } from "@web3auth/ethereum-provider";
```
We import the necessary libraries and set up configuration variables for Web3Auth and the Ethereum chain (Base Chain) we are using.

### Chain 	Config
```javascript
const clientId = "ID from Web3Auth Dashboard";
const chainConfig = {
  chainNamespace: "eip155",
  chainId: "0x14A34", // hex of 84532
  rpcTarget: "https://sepolia.base.org",
  displayName: "Base Sepolia",
  blockExplorerUrl: "https://sepolia-explorer.base.org",
  ticker: "ETH",
  tickerName: "ETH",
};
```
The clientId is the ID from our application on the Web3Auth Dev dashboard. The chainConfig contains the configuration object details for the Base Sepolia network. This part changes depending on the chain we want to deploy on. 


### Contract ABI and Bytecode
```javascript
const contractABI = [
  { inputs: [{ internalType: "string", name: "initMessage", type: "string" }], stateMutability: "nonpayable", type: "constructor" },
  { inputs: [], name: "message", outputs: [{ internalType: "string", name: "", type: "string" }], stateMutability: "view", type: "function" },
  { inputs: [{ internalType: "string", name: "newMessage", type: "string" }], name: "update", outputs: [], stateMutability: "nonpayable", type: "function" },
];

const contractByteCode = “Your contract Bytecode”
```
We define the ABI (Application Binary Interface) and bytecode for our smart contract. This example uses a simple contract with a constructor, a message function, and an update function. Also we have the Bytecode of the contract. You can use the default values provided in the Web3Auth documentation but you can also choose to deploy the contract yourself and then add the ABI and Bytecode to your code.


### Main Component
```javascript
const App = () => {
  const [web3auth, setWeb3auth] = useState(null);
  const [provider, setProvider] = useState(null);
  const [contract, setContract] = useState(null);
  const [message, setMessage] = useState("");
  const [newMessage, setNewMessage] = useState("");


  useEffect(() => {
    const init = async () => {
      try {
        const web3auth = new Web3AuthNoModal({
          clientId,
          chainConfig,
          web3AuthNetwork: "sapphire_devnet",
        });


        const privateKeyProvider = new EthereumPrivateKeyProvider({
          config: { chainConfig },
        });


        const openloginAdapter = new OpenloginAdapter({
          adapterSettings: {
            clientId,
            uxMode: "popup",
            loginConfig: {
              jwt: {
                verifier: "app-test",
                typeOfLogin: "jwt",
                clientId: "YOUR_AUTH0_CLIENT_ID", 
              },
            },
          },
          privateKeyProvider,
        });


        web3auth.configureAdapter(openloginAdapter);
        setWeb3auth(web3auth);


        await web3auth.init();
      } catch (error) {
        console.error(error);
      }
    };


    init();
  }, [])}
  export default App;
```
The main App component, using hooks to manage state and side effects. We also initialize and instantiate the Web3Auth SDK. 


### Login Function
```javascript
const login = async () => {
    if (!web3auth) {
      console.log("web3auth not initialized yet");
      return;
    }
    const web3authProvider = await web3auth.connectTo(WALLET_ADAPTERS.OPENLOGIN, {
      loginProvider: "jwt",
      extraLoginOptions: {
        domain: "https://Your-Auth0-domain-url",
        verifierIdField: "sub",
      },
    });
    setProvider(new ethers.BrowserProvider(web3authProvider));
  };
```
This is the asynchronous function to log in using Web3Auth and set the provider. The domain name and other details are obtained from the Web3Auth dev dashboard.  


### Get User Info Function
```javascript
const getUserInfo = async () => {
    if (!web3auth) {
      console.log("web3auth not initialized yet");
      return;
    }
    const user = await web3auth.getUserInfo();
    console.log(user);
  };
```
This is an asynchronous function to get and log the user's information from Web3Auth.

### Logout function 
```javascript
const logout = async () => {
    if (!web3auth) {
      console.log("web3auth not initialized yet");
      return;
    }
    await web3auth.logout();
    setProvider(null);
  };
```
This is an asynchronous function to log the user out of Web3Auth and reset the provider state.

### Deploy Contract Function
```javascript
  const deployContract = async () => {
    if (!provider) {
      console.log("provider not initialized yet");
      return;
    }
    try {
      const signer = await provider.getSigner();
      const factory = new ethers.ContractFactory(contractABI, contractByteCode, signer);
      const contract = await factory.deploy("Hello World!");
      await contract.waitForDeployment();
      setContract(contract);
      console.log("Contract deployed to:", await contract.getAddress());
    } catch (error) {
      console.error("Error deploying contract:", error.message);
      if (error.message.includes("insufficient funds")) {
        console.log("Please make sure you have enough test ETH in your wallet.");
      }
    }
  };
```
This is an asynchronous function that deploys a smart contract to the blockchain network that was specified, in this case, the Base Chain network and then sets the contract state.

### Read Contract Function
```javascript
const readContract = async () => {
    if (!contract) {
      console.log("Contract not deployed yet");
      return;
    }
    const message = await contract.message();
    setMessage(message);
  };
```
This is an asynchronous function to read the current message from the smart contract and set the message state.

### Write Contract Function
```javascript
const writeContract = async () => {
    if (!contract) {
      console.log("Contract not deployed yet");
      return;
    }
    const tx = await contract.update(newMessage);
    await tx.wait();
    console.log("Message updated");
    setNewMessage("");
  }
```
This is an asynchronous function to update the message in the smart contract and reset the newMessage state.

### Check Balance Function
```javascript
 const checkBalance = async () => { if (!provider){ 
    console.log("Provider not initialized yet"); 
    return;
     } try { 
    const signer = await provider.getSigner();
    const address = await signer.getAddress();
    console.log("Wallet address:", address);
    const network = await provider.getNetwork();
    console.log("Connected to network:", network.name, "Chain ID:", network.chainId);
    const balance = await provider.getBalance(address);
    console.log("Balance:", ethers.formatEther(balance), "ETH"); }
      catch (error) { 
        console.error("Error checking balance:", error);
       } };
```
This is an asynchronous function to check and log the balance of the connected wallet.

### Render the Component
```javascript
return (
    <main>
      <h1>Web3Auth + Smart Contract Interaction</h1>
      {!provider ? (
        <button onClick={login}>Login</button>
      ) : (
        <>
          <button onClick={getUserInfo}>Get User Info</button>
          <button onClick={logout}>Logout</button>
          <button onClick={deployContract}>Deploy Contract</button>
          <button onClick={checkBalance}>Check Balance</button>


          {contract && (
            <>
              <button onClick={readContract}>Read Contract</button>
             
              <div>
              <input
                type="text"
                value={newMessage}
                onChange={(e) => setNewMessage(e.target.value)}
                placeholder="Enter new message"
              />
              <button onClick={writeContract}>Update Message</button>
              </div>
            </>
          )}
          <p>Current Message: {message}</p>
        </>
      )}
    </main>
  );
```
This renders the UI with buttons to trigger the functions and display the message. There is conditional rendering based on whether the provider is set. Buttons to trigger login, get user info, logout, deploy contract, check balance, read contract, and update message in the smart contract. Then an input field for the message\. 

### Full code
```javascript

import React, { useState, useEffect } from 'react';
import { ethers } from 'ethers';
import { WALLET_ADAPTERS } from "@web3auth/base";
import { Web3AuthNoModal } from "@web3auth/no-modal";
import { OpenloginAdapter } from "@web3auth/openlogin-adapter";
import { EthereumPrivateKeyProvider } from "@web3auth/ethereum-provider";

const clientId = "ID from Web3Auth Dashboard";
const chainConfig = {
  chainNamespace: "eip155",
  chainId: "0x14A34", // hex of 84532
  rpcTarget: "https://sepolia.base.org",
  displayName: "Base Sepolia",
  blockExplorerUrl: "https://sepolia-explorer.base.org",
  ticker: "ETH",
  tickerName: "ETH",
};

const contractABI = [
  { inputs: [{ internalType: "string", name: "initMessage", type: "string" }], stateMutability: "nonpayable", type: "constructor" },
  { inputs: [], name: "message", outputs: [{ internalType: "string", name: "", type: "string" }], stateMutability: "view", type: "function" },
  { inputs: [{ internalType: "string", name: "newMessage", type: "string" }], name: "update", outputs: [], stateMutability: "nonpayable", type: "function" },
];

const contractByteCode = “Your contract Bytecode”;

const App = () => {
  const [web3auth, setWeb3auth] = useState(null);
  const [provider, setProvider] = useState(null);
  const [contract, setContract] = useState(null);
  const [message, setMessage] = useState("");
  const [newMessage, setNewMessage] = useState("");

  useEffect(() => {
    const init = async () => {
      try {
        const web3auth = new Web3AuthNoModal({
          clientId,
          chainConfig,
          web3AuthNetwork: "sapphire_devnet",
        });

        const privateKeyProvider = new EthereumPrivateKeyProvider({
          config: { chainConfig },
        });

        const openloginAdapter = new OpenloginAdapter({
          adapterSettings: {
            clientId,
            uxMode: "popup",
            loginConfig: {
              jwt: {
                verifier: "app-test",
                typeOfLogin: "jwt",
                clientId: "YOUR_AUTH0_CLIENT_ID",
              },
            },
          },
          privateKeyProvider,
        });

        web3auth.configureAdapter(openloginAdapter);

        setWeb3auth(web3auth);

        await web3auth.init();
      } catch (error) {
        console.error(error);
      }
    };

    init();
  }, []);

  const login = async () => {
    if (!web3auth) {
      console.log("web3auth not initialized yet");
      return;
    }
    const web3authProvider = await web3auth.connectTo(WALLET_ADAPTERS.OPENLOGIN, {
      loginProvider: "jwt",
      extraLoginOptions: {
        domain: "https://Your-Auth0-domain-url",
        verifierIdField: "sub",
      },
    });
    setProvider(new ethers.BrowserProvider(web3authProvider));
  };

  const getUserInfo = async () => {
    if (!web3auth) {
      console.log("web3auth not initialized yet");
      return;
    }
    const user = await web3auth.getUserInfo();
    console.log(user);
  };

  const logout = async () => {
    if (!web3auth) {
      console.log("web3auth not initialized yet");
      return;
    }
    await web3auth.logout();
    setProvider(null);
  };

  const deployContract = async () => {
    if (!provider) {
      console.log("provider not initialized yet");
      return;
    }
    try {
      const signer = await provider.getSigner();
      const factory = new ethers.ContractFactory(contractABI, contractByteCode, signer);
      const contract = await factory.deploy("Hello World!");
      await contract.waitForDeployment();
      setContract(contract);
      console.log("Contract deployed to:", await contract.getAddress());
    } catch (error) {
      console.error("Error deploying contract:", error.message);
      if (error.message.includes("insufficient funds")) {
        console.log("Please make sure you have enough test ETH in your wallet.");
      }
    }
  };
  
  const readContract = async () => {
    if (!contract) {
      console.log("Contract not deployed yet");
      return;
    }
    const message = await contract.message();
    setMessage(message);
  };

  const writeContract = async () => {
    if (!contract) {
      console.log("Contract not deployed yet");
      return;
    }
    const tx = await contract.update(newMessage);
    await tx.wait();
    console.log("Message updated");
    setNewMessage("");
  };

  const checkBalance = async () => { if (!provider)
     { console.log("Provider not initialized yet"); return; 
     } try { const signer = await provider.getSigner(); 
      const address = await signer.getAddress(); 
      console.log("Wallet address:", address); 
      const network = await provider.getNetwork(); 
      console.log("Connected to network:", network.name, "Chain ID:", network.chainId); 
      const balance = await provider.getBalance(address); 
      console.log("Balance:", ethers.formatEther(balance), "ETH"); } 
      catch (error) { console.error("Error checking balance:", error);
       } };

  return (
    <main>
      <h1>Web3Auth + Smart Contract Interaction</h1>
      {!provider ? (
        <button onClick={login}>Login</button>
      ) : (
        <>
          <button onClick={getUserInfo}>Get User Info</button>
          <button onClick={logout}>Logout</button>
          <button onClick={deployContract}>Deploy Contract</button>
          <button onClick={checkBalance}>Check Balance</button>

          {contract && (
            <>
              <button onClick={readContract}>Read Contract</button>
              
              <div>
              <input
                type="text"
                value={newMessage}
                onChange={(e) => setNewMessage(e.target.value)}
                placeholder="Enter new message"
              />
              <button onClick={writeContract}>Update Message</button>
              </div>
            </>
          )}
          <p>Current Message: {message}</p>
        </>
      )}
    </main>
  );
}
export default App;
```
## Runing the Application
- Start the development server.
```
npm start
```
- Open http://localhost:3000 in your browser
- Click "Login" and authenticate with Apple
- Click "Check Balance" and confirm your ETH balance and wallet address
- Deploy the contract (ensure you have test ETH in your wallet)
- Read from and write to the contract
- Check your wallet balance

### Conclusion
This example demonstrates how to integrate Web3Auth with a React app for authenticating users, deploying and interacting with an Ethereum smart contract, and checking wallet balances. Customize the configuration and contract details as needed for your specific use case.

