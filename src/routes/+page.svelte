<script lang="ts">
  // State variables for the inputs
  let callsign = $state<string>('');
  let address = $state<string>('');
  let error = $state<string>('');

  // Character validation for callsigns
  const validChars = /^[A-Z0-9\-/. ]*$/;

  // Convert callsign to hex address
  function callsignToAddress(): void {
    if (!callsign) {
      address = '';
      error = '';
      return;
    }

    if (callsign.length > 9) {
      error = 'Callsign must be 9 characters or less';
      return;
    }

    if (!validChars.test(callsign)) {
      error = 'Invalid characters in callsign. Use A-Z, 0-9, -, /, or .';
      return;
    }

    error = '';
    const encoded = encodeCallsignBase40(callsign);
    address = encoded.toString(16).toUpperCase().padStart(12, '0');
  }

  // Convert hex address to callsign
  function addressToCallsign(): void {
    if (!address) {
      callsign = '';
      error = '';
      return;
    }

    error = '';

    try {
      const hexRegex = /^[0-9A-Fa-f]{1,12}$/;
      if (!hexRegex.test(address)) {
        error = 'Address must be a valid hexadecimal value (12 digits or less)';
        return;
      }

      const encoded = BigInt('0x' + address.padStart(12, '0'));
      
      // Check if the address is out of range
      if (encoded === 0n) {
        error = 'Address 0x000000000000 is invalid (reserved)';
        return;
      }
      
      if (encoded === 0xFFFFFFFFFFFFn) {
        callsign = 'BROADCAST';
        return;
      }
      
      if (encoded >= 262144000000000n && encoded <= 0xFFFFFFFFFFFEn) {
        error = 'Address is in reserved range';
        return;
      }
      
      if (encoded > 262143999999999n) {
        error = 'Address is out of valid range';
        return;
      }

      callsign = decodeCallsignBase40(encoded);
    } catch (e) {
      error = `Error: ${e instanceof Error ? e.message : String(e)}`;
    }
  }

  // Handle callsign input
  function handleCallsignInput(): void {
    error = '';
    callsignToAddress();
  }

  // Handle address input
  function handleAddressInput(): void {
    error = '';
    addressToCallsign();
  }

  // Encode callsign using base40 as specified in the M17 protocol
  function encodeCallsignBase40(callsign: string): bigint {
    let encoded = 0n;
    for (let i = callsign.length - 1; i >= 0; i--) {
      const char = callsign[i];
      encoded *= 40n;
      
      if (char >= 'A' && char <= 'Z') {
        encoded += BigInt(char.charCodeAt(0) - 'A'.charCodeAt(0) + 1);
      } else if (char >= '0' && char <= '9') {
        encoded += BigInt(char.charCodeAt(0) - '0'.charCodeAt(0) + 27);
      } else if (char === '-') {
        encoded += 37n;
      } else if (char === '/') {
        encoded += 38n;
      } else if (char === '.') {
        encoded += 39n;
      } else {
        // Space or invalid character
        encoded += 0n;
      }
    }
    return encoded;
  }

  function decodeCallsignBase40(encoded: bigint): string {
    if (encoded >= 262144000000000n) {
      return '';
    }
    
    const baseChars = " ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789-/.";
    let reversedResult = '';
    
    while (encoded > 0n) {
      const remainder = Number(encoded % 40n);
      reversedResult += baseChars[remainder];
      encoded = encoded / 40n;
    }

    return reversedResult;
  }
</script>

<main>
  <h1>M17 Protocol Callsign Converter</h1>
  
  <div class="converter-container">
    <div class="player-card dark-theme">
      <div class="card-header">
        <img id="playerLogo" src="favicon.png" alt="M17 Logo">
        <div class="card-title">Callsign Converter</div>
      </div>
      
      <div class="card-body">
        <div class="input-group">
          <label for="callsign" class="label">Callsign (9 chars max):</label>
          <input 
            id="callsign"
            type="text"
            bind:value={callsign}
            oninput={handleCallsignInput}
            placeholder="Enter callsign (e.g. W1AW)"
            maxlength="9"
            class="converter-input"
          />
        </div>
        
        <div class="input-group">
          <label for="address" class="label">Hex Address (48-bit):</label>
          <input 
            id="address"
            type="text"
            bind:value={address}
            oninput={handleAddressInput}
            placeholder="Enter hex address (e.g. 000000BC614E)"
            class="converter-input"
          />
        </div>
        
        {#if error}
          <div class="error-message">{error}</div>
        {/if}
      </div>
      
      <div class="card-footer">
        <div class="status-container">
          <span class={address ? 'status-connected' : 'status-disconnected'}>
            {address ? 'Valid conversion' : 'Waiting for input'}
          </span>
        </div>
        <a href="https://github.com/OE3ANC/m17web-callculator" class="github-link" target="_blank" rel="noopener noreferrer">
          m17web-callculator
        </a>
      </div>
    </div>
  </div>
</main>

<style>
  @import url('https://fonts.googleapis.com/css2?family=Roboto:wght@300;400;500;700&display=swap');

  main {
    margin: 0 auto;
    padding: 2rem;
    font-family: 'Roboto', sans-serif;
    background: #ffffff;
  }
  
  h1 {
    color: #333;
    text-align: center;
    margin-bottom: 2rem;
    font-weight: 500;
  }
  
  .converter-container {
    display: flex;
    flex-wrap: wrap;
    justify-content: center;
    align-items: flex-start;
    gap: 20px;
    justify-content: center;
  }
  
  .player-card {
    min-width: 400px;
    width: auto;
    border-radius: 12px;
    overflow: hidden;
    box-shadow: 0 4px 20px rgba(0,0,0,0.15);
    margin: 15px;
    transition: all 1s ease;
    font-family: 'Roboto', sans-serif;
    flex: 0 1 auto;
    max-width: 48%;
  }
  
  .player-card:hover {
    transform: translateY(-2.5px);
    box-shadow: 0 8px 25px rgba(0,0,0,0.2);
  }
  
  .dark-theme {
    background: repeating-linear-gradient(
      -55deg,
      #353535,
      #353535 10px,
      #303030 10px,
      #303030 20px
    );
    color: #ffffff;
    transition: all 1s ease;
    border: 2.5px solid rgb(34, 34, 34);
  }
  
  .card-header {
    padding: 15px;
    display: flex;
    align-items: center;
    border-bottom: 1px solid rgba(255, 255, 255, 0.05);
    transition: all 1s ease;
    background: rgb(34, 34, 34);
  }
  
  #playerLogo {
    max-width: 35px;
    max-height: 35px;
    width: auto;
    height: auto;
    margin-right: 10px;
    border-radius: 50%;
  }
  
  .card-title {
    font-size: 18px;
    font-weight: 500;
  }
  
  .card-body {
    padding: 20px 15px;
  }
  
  .input-group {
    margin-bottom: 15px;
  }
  
  .label {
    font-size: 14px;
    opacity: 0.8;
    display: block;
    margin-bottom: 5px;
  }
  
  .converter-input {
    width: 95%;
    padding: 10px;
    background: rgba(255, 255, 255, 0.1);
    border: 1px solid rgba(255, 255, 255, 0.2);
    border-radius: 5px;
    color: white;
    font-family: monospace;
    font-size: 14px;
    outline: none;
    transition: all 0.3s ease;
  }
  
  .converter-input:focus {
    border-color: #4285F4;
    box-shadow: 0 0 0 2px rgba(66, 133, 244, 0.3);
  }
  
  .converter-input::placeholder {
    color: rgba(255, 255, 255, 0.4);
  }
  
  .error-message {
    background: rgba(244, 67, 54, 0.2);
    border-left: 3px solid #F44336;
    padding: 10px;
    border-radius: 4px;
    font-size: 14px;
    margin-top: 15px;
  }
  
  .card-footer {
    padding: 15px;
    display: flex;
    align-items: center;
    justify-content: space-between;
    border-top: 1px solid rgba(255,255,255,0.05);
    background: rgb(34, 34, 34);
  }
  
  .status-container {
    display: flex;
    justify-content: flex-start;
    font-size: 12px;
  }
  
  .status-connected {
    color: #4CAF50;
  }
  
  .status-disconnected {
    color: #9E9E9E;
  }

  .github-link {
    color: #64B5F6;
    text-decoration: none;
    font-size: 14px;
    font-weight: 500;
    transition: all 0.3s ease;
  }

  .github-link:hover {
    color: #90CAF9;
    text-decoration: underline;
  }

</style>