# Simple Blockchain in Go

This project demonstrates how to create a simple blockchain in less than 200 lines of Go. It is based on the tutorial [Code your own blockchain in less than 200 lines of Go](https://mycoralhealth.medium.com/code-your-own-blockchain-in-less-than-200-lines-of-go-e296282bcffc) by Coral Health.

## Overview

A blockchain is a decentralized, distributed ledger that records transactions across many computers in such a way that the registered transactions cannot be altered retroactively. This project implements a basic blockchain with the following functionalities:
- Creating a genesis block
- Adding new blocks
- Validating the blockchain

## Project Structure

- `main.go`: The main Go file containing all the logic for the blockchain.

## Prerequisites

- [Go](https://golang.org/doc/install) installed (version 1.13+)
- [Go modules](https://blog.golang.org/using-go-modules) enabled
- Environment variable configuration using `.env` file

## Installation

1. Clone the repository:
    ```sh
    git clone https://github.com/RMATLee/Go-Blockchain.git
    cd Go-Blockchain
    ```

2. Create a `.env` file in the root of the project and add the following line:
    ```sh
    PORT=8080
    ```

3. Install dependencies:
    ```sh
    go mod tidy
    ```

## Running the Application

To run the blockchain application, use the following command:
```sh
go run main.go
```

This will start an HTTP server listening on the port specified in your `.env` file.

## API Endpoints

### GET /

Returns the current state of the blockchain.

**Example Request:**
```sh
curl http://localhost:8080/
```

**Example Response:**
```json
[
  {
    "Index": 0,
    "Timestamp": "2023-01-01 00:00:00 +0000 UTC",
    "BPM": 0,
    "Hash": "d2d2d2...",
    "PrevHash": ""
  }
]
```

### POST /

Adds a new block to the blockchain with the provided BPM (beats per minute).

**Example Request:**
```sh
curl -X POST -H "Content-Type: application/json" -d '{"BPM":70}' http://localhost:8080/
```

**Example Response:**
```json
{
  "Index": 1,
  "Timestamp": "2023-01-01 00:01:00 +0000 UTC",
  "BPM": 70,
  "Hash": "e3e3e3...",
  "PrevHash": "d2d2d2..."
}
```

## Code Explanation

### Block Struct
```go
type Block struct {
    Index     int    // Position of the data record in the blockchain
    Timestamp string // Time the data is written
    BPM       int    // Beats per minute (heart rate data)
    Hash      string // SHA256 identifier representing this data record
    PrevHash  string // SHA256 identifier of the previous data record
}
```

### Main Function
The `main` function loads environment variables, creates the genesis block, and starts the HTTP server.
```go
func main() {
    err := godotenv.Load()
    if err != nil {
        log.Fatal(err)
    }

    go func() {
        t := time.Now()
        genesisBlock := Block{}
        genesisBlock = Block{0, t.String(), 0, calculateHash(genesisBlock), ""}
        spew.Dump(genesisBlock)

        mutex.Lock()
        Blockchain = append(Blockchain, genesisBlock)
        mutex.Unlock()
    }()
    log.Fatal(run())
}
```

### HTTP Server
The `run` function initializes and starts the HTTP server.
```go
func run() error {
    mux := makeMuxRouter()
    httpPort := os.Getenv("PORT")
    log.Println("HTTP Server Listening on port :", httpPort)
    s := &http.Server{
        Addr:           ":" + httpPort,
        Handler:        mux,
        ReadTimeout:    10 * time.Second,
        WriteTimeout:   10 * time.Second,
        MaxHeaderBytes: 1 << 20,
    }

    if err := s.ListenAndServe(); err != nil {
        return err
    }

    return nil
}
```

### Handlers
The handlers manage HTTP requests to get the blockchain and add new blocks.
```go
func handleGetBlockchain(w http.ResponseWriter, r *http.Request) {
    bytes, err := json.MarshalIndent(Blockchain, "", "  ")
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    io.WriteString(w, string(bytes))
}

func handleWriteBlock(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    var msg Message

    decoder := json.NewDecoder(r.Body)
    if err := decoder.Decode(&msg); err != nil {
        respondWithJSON(w, http.StatusBadRequest, r.Body)
        return
    }
    defer r.Body.Close()

    mutex.Lock()
    prevBlock := Blockchain[len(Blockchain)-1]
    newBlock := generateBlock(prevBlock, msg.BPM)

    if isBlockValid(newBlock, prevBlock) {
        Blockchain = append(Blockchain, newBlock)
        spew.Dump(Blockchain)
    }
    mutex.Unlock()

    respondWithJSON(w, http.StatusCreated, newBlock)
}
```

## License

This project is licensed under the MIT License.
