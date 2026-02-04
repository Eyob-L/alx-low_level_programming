Here's a clean, simplified version that looks like something you'd write yourself:

```go
package main

import (
	"bufio"
	"fmt"
	"os"
	"strconv"
	"strings"
)

// Colors for the terminal output
const green = "\u001B[32m"
const yellow = "\u001B[33m"
const white = "\u001B[37m"
const reset = "\u001B[0m"

func main() {
	// Check if we got a word number
	if len(os.Args) < 2 {
		return
	}

	// Get which word to use from the list
	wordNum, err := strconv.Atoi(os.Args[1])
	if err != nil {
		return
	}

	// Read the word list
	words := loadWords("wordle-words.txt")
	if wordNum >= len(words) {
		return
	}

	// Get username
	fmt.Print("Enter your username: ")
	reader := bufio.NewReader(os.Stdin)
	username, _ := reader.ReadString('\n')
	username = strings.TrimSpace(username)

	// Start the game
	secret := words[wordNum]
	playGame(username, secret)
}

// Load words from file
func loadWords(filename string) []string {
	file, err := os.Open(filename)
	if err != nil {
		return []string{}
	}
	defer file.Close()

	var words []string
	scanner := bufio.NewScanner(file)
	for scanner.Scan() {
		word := strings.ToUpper(strings.TrimSpace(scanner.Text()))
		if len(word) == 5 {
			words = append(words, word)
		}
	}
	return words
}

// Main game logic
func playGame(username, secret string) {
	fmt.Println("Welcome to Wordle! Guess the 5-letter word")

	scanner := bufio.NewScanner(os.Stdin)
	guessedLetters := make(map[byte]bool)

	for attempt := 1; attempt <= 6; attempt++ {
		fmt.Print("Enter your guess: ")
		scanner.Scan()
		guess := strings.ToUpper(strings.TrimSpace(scanner.Text()))

		// Check guess length
		if len(guess) != 5 {
			fmt.Println("Please enter a 5-letter word")
			attempt-- // Don't count invalid guesses
			continue
		}

		// Show colored feedback
		fmt.Print("Feedback: ")
		for i := 0; i < 5; i++ {
			letter := guess[i]
			
			// Green for correct position
			if letter == secret[i] {
				fmt.Printf("%s%c%s", green, letter, reset)
			} else if strings.Contains(secret, string(letter)) {
				// Yellow for correct letter wrong position
				fmt.Printf("%s%c%s", yellow, letter, reset)
			} else {
				// White for wrong letter
				fmt.Printf("%s%c%s", white, letter, reset)
				guessedLetters[letter] = true
			}
		}
		fmt.Println()

		// Show letters that haven't been guessed
		fmt.Print("Remaining letters: ")
		for c := 'A'; c <= 'Z'; c++ {
			if guessedLetters[byte(c)] {
				fmt.Printf("%c ", c) // uppercase for guessed
			} else {
				fmt.Printf("%c ", c+32) // lowercase for not guessed
			}
		}
		fmt.Println()

		// Check if won
		if guess == secret {
			fmt.Printf("Attempts remaining: %d\n", 6-attempt)
			saveResult(username, secret, attempt, "win")
			askForStats(username)
			return
		}

		// Show attempts left
		fmt.Printf("Attempts remaining: %d\n", 6-attempt)
	}

	// If we get here, player lost
	fmt.Printf("The word was: %s\n", secret)
	saveResult(username, secret, 6, "loss")
	askForStats(username)
}

// Save game result to file
func saveResult(username, word string, attempts int, result string) {
	file, _ := os.OpenFile("stats.csv", os.O_APPEND|os.O_CREATE|os.O_WRONLY, 0644)
	fmt.Fprintf(file, "%s,%s,%d,%s\n", username, word, attempts, result)
	file.Close()
}

// Ask if player wants to see stats
func askForStats(username string) {
	fmt.Print("\nDo you want to see your stats? (yes/no): ")
	scanner := bufio.NewScanner(os.Stdin)
	scanner.Scan()
	
	if scanner.Text() != "yes" {
		return
	}

	// Try to read stats file
	file, err := os.Open("stats.csv")
	if err != nil {
		return
	}
	defer file.Close()

	// Count games
	games, wins, totalAttempts := 0, 0, 0
	scanner = bufio.NewScanner(file)
	for scanner.Scan() {
		parts := strings.Split(scanner.Text(), ",")
		if len(parts) >= 4 && parts[0] == username {
			games++
			if parts[3] == "win" {
				wins++
			}
			att, _ := strconv.Atoi(parts[2])
			totalAttempts += att
		}
	}

	// Show stats
	fmt.Printf("\nStats for %s:\n", username)
	fmt.Printf("  Games played: %d\n", games)
	fmt.Printf("  Games won: %d\n", wins)
	
	avg := 0.0
	if games > 0 {
		avg = float64(totalAttempts) / float64(games)
	}
	fmt.Printf("  Average attempts per game: %.1f\n", avg)
	
	// Wait before exiting
	fmt.Print("\nPress Enter to exit...")
	bufio.NewReader(os.Stdin).ReadBytes('\n')
}
```
