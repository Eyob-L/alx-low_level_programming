package main

import (
	"bufio"
	"fmt"
	"os"
	"strconv"
	"strings"
)

const (
	Green  = "\u001B[32m"
	Yellow = "\u001B[33m"
	White  = "\u001B[37m"
	Reset  = "\u001B[0m"
)

func main() {
	if len(os.Args) < 2 {
		return
	}

	index, err := strconv.Atoi(os.Args[1])
	if err != nil {
		return
	}

	words := readWords("wordle-words.txt")
	if index >= len(words) {
		return
	}
	secret := words[index]

	fmt.Print("Enter your username: ")
	scanner := bufio.NewScanner(os.Stdin)
	scanner.Scan()
	username := scanner.Text()

	fmt.Println("Welcome to Wordle! Guess the 5-letter word")
	attempts := 0
	guessed := make(map[byte]bool)

	for attempts < 6 {
		fmt.Print("Enter your guess: ")
		scanner.Scan()
		guess := strings.ToUpper(scanner.Text())

		if len(guess) != 5 {
			fmt.Println("Please enter a 5-letter word")
			continue
		}

		attempts++
		feedback := make([]string, 5)
		secretCopy := secret

		// Green pass
		for i := 0; i < 5; i++ {
			if guess[i] == secret[i] {
				feedback[i] = Green + string(guess[i]) + Reset
				secretCopy = secretCopy[:i] + " " + secretCopy[i+1:]
			}
		}

		// Yellow/White pass
		for i := 0; i < 5; i++ {
			if feedback[i] != "" {
				continue
			}
			if idx := strings.IndexByte(secretCopy, guess[i]); idx != -1 {
				feedback[i] = Yellow + string(guess[i]) + Reset
				secretCopy = secretCopy[:idx] + " " + secretCopy[idx+1:]
			} else {
				feedback[i] = White + string(guess[i]) + Reset
				guessed[guess[i]] = true
			}
		}

		fmt.Print("Feedback: ")
		for _, f := range feedback {
			fmt.Print(f)
		}
		fmt.Println()

		// Remaining letters
		fmt.Print("Remaining letters: ")
		for c := 'A'; c <= 'Z'; c++ {
			if guessed[byte(c)] {
				fmt.Printf("%c ", c)
			} else {
				fmt.Printf("%c ", c+32)
			}
		}
		fmt.Println()

		if guess == secret {
			fmt.Printf("Attempts remaining: %d\n", 6-attempts)
			saveStats(username, secret, attempts, "win")
			askStats(username)
			return
		}

		fmt.Printf("Attempts remaining: %d\n", 6-attempts)
	}

	fmt.Printf("The word was: %s\n", secret)
	saveStats(username, secret, 6, "loss")
	askStats(username)
}

func readWords(filename string) []string {
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

func saveStats(username, word string, attempts int, result string) {
	file, _ := os.OpenFile("stats.csv", os.O_APPEND|os.O_CREATE|os.O_WRONLY, 0644)
	fmt.Fprintf(file, "%s,%s,%d,%s\n", username, word, attempts, result)
	file.Close()
}

func askStats(username string) {
	fmt.Print("\nDo you want to see your stats? (yes/no): ")
	scanner := bufio.NewScanner(os.Stdin)
	scanner.Scan()
	if scanner.Text() != "yes" {
		return
	}

	file, err := os.Open("stats.csv")
	if err != nil {
		return
	}
	defer file.Close()

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

	fmt.Printf("\nStats for %s:\n", username)
	fmt.Printf("  Games played: %d\n", games)
	fmt.Printf("  Games won: %d\n", wins)
	avg := 0.0
	if games > 0 {
		avg = float64(totalAttempts) / float64(games)
	}
	fmt.Printf("  Average attempts per game: %.1f\n", avg)
	fmt.Print("\nPress Enter to exit...")
	bufio.NewReader(os.Stdin).ReadBytes('\n')
}
