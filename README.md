# golang-new-relic-transaction-segment-sample

This repository is demo how to use new relic and segment to profile golang application

## implementation

```golang
package main

import (
	"errors"
	"fmt"
	"log"
	"math/rand"
	"os"
	"time"

	"github.com/leetcode-golang-classroom/golang-new-relic-transaction-segment-sample/internal/config"
	"github.com/newrelic/go-agent/v3/newrelic"
)

var (
	nrApp    *newrelic.Application
	nrErr    error
	randomer *rand.Rand
)

func main() {
	nrApp, nrErr = newrelic.NewApplication(
		newrelic.ConfigAppName(config.AppConfig.AppName),
		newrelic.ConfigLicense(config.AppConfig.NewRelicLicenseKey),
		newrelic.ConfigDebugLogger(os.Stdout),

		// Workshop > Additional configuration via a config function
		// https://docs.newrelic.com/docs/apm/agents/go-agent/configuration/go-agent-configuration
		func(cfg *newrelic.Config) {
			cfg.Enabled = true
			cfg.DistributedTracer.Enabled = true
			cfg.Labels = map[string]string{
				"Env":      "Dev",
				"Function": "Greetings",
				"Platform": "My Machine",
				"Team":     "API",
			}
		},
	)
	if nrErr != nil {
		log.Fatalln("unable to start NR instrumentation -", nrErr)
	}

	if waitErr := nrApp.WaitForConnection(5 * time.Second); waitErr != nil {
		log.Fatal("nrApp.WaitForConnection failed")
	}

	// Request a greeting message
	message, err := Hello("Tony Stark")

	if err != nil {
		log.Fatal(err)
	}
	fmt.Println(message)

	nrApp.Shutdown(5 * time.Second)
}

// init set initial values for variables used in the function
func init() {
	randomer = rand.New(rand.NewSource(time.Now().UnixNano()))
}

// Hello returns a greeting for the name person
func Hello(name string) (string, error) {
	// Monitor a transaction
	nrTxnTracer := nrApp.StartTransaction("Hello")
	defer nrTxnTracer.End()

	// If no name was given, return an error with a message
	if name == "" {
		return name, errors.New("empty name")
	}

	// Create a message using a random format
	message := fmt.Sprintf(randomFormat(), name)

	// custom attributes by using method in a transaction
	nrTxnTracer.AddAttribute("message", message)
	return message, nil
}

// randomFormat returns one of a set of greeting messages
func randomFormat() string {
	// Set up transaction to trace
	nrTxnTracer := nrApp.StartTransaction("randomFormat")
	defer nrTxnTracer.End()

	randomDelayOuter := randomer.Intn(40)
	time.Sleep(time.Duration(randomDelayOuter) * time.Microsecond)

	// create a segment for Formats
	nrSegment := nrTxnTracer.StartSegment("Formats")

	// Random sleep to simulate delays
	randomDelayInner := randomer.Intn(80)
	time.Sleep(time.Duration(randomDelayInner) * time.Microsecond)

	// A slice of message formats.
	formats := []string{
		"Hi, %v. Welcome!",
		"Great to see you, %v!",
		"Good day, %v! Well met!",
		"%v! Hi there!",
		"Greetings %v!",
		"Hello there, %v!",
	}

	// end a segment
	nrSegment.End()
	// Return a randomly selected message format by specifying
	// a random index for the slice of formats.
	return formats[randomer.Intn(len(formats))]
}

```