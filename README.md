# simple-godog-example

```
package main_test

import (
	"context"
	"fmt"
	"net/http"
	"net/http/httptest"

	"github.com/cucumber/godog"
	"github.com/labstack/echo/v4"
	"github.com/stretchr/testify/assert"
)

type testFeature struct {
	resp *httptest.ResponseRecorder
}

func (tf *testFeature) aRunningServer() error {
	// create a new Echo instance and add a Healthcheck endpoint
	e := echo.New()
	e.GET("/healthcheck", func(c echo.Context) error {
		return c.String(http.StatusOK, "OK")
	})

	// start a test HTTP server
	tf.resp = httptest.NewRecorder()
	go func() {
		if err := e.Start(":8080"); err != nil {
			panic(err)
		}
	}()
	return nil
}

func (tf *testFeature) iSendARequestToTheHealthcheckEndpoint() error {
	// send a GET request to the Healthcheck endpoint
	req, err := http.NewRequestWithContext(context.Background(), "GET", "/healthcheck", nil)
	if err != nil {
		return err
	}
	resp := tf.resp
	e := echo.New()
	e.ServeHTTP(resp, req)
	return nil
}

func (tf *testFeature) theResponseCodeShouldBe(code int) error {
	// check that the response code matches the expected code
	assert.Equal(fmt.Sprintf("%d %s", code, http.StatusText(code)), tf.resp.Header().Get("HTTP Status"))
	return nil
}

func (tf *testFeature) theResponseBodyShouldBe(body string) error {
	// check that the response body matches the expected body
	assert.Equal(body, tf.resp.Body.String())
	return nil
}

func InitializeScenario(ctx *godog.ScenarioContext) {
	tf := &testFeature{}

	ctx.BeforeScenario(func(*godog.Scenario) {
		// start the server before each scenario
		if err := tf.aRunningServer(); err != nil {
			panic(err)
		}
	})

	ctx.Step(`^I send a request to the healthcheck endpoint$`, tf.iSendARequestToTheHealthcheckEndpoint)
	ctx.Step(`^the response code should be (\d+)$`, tf.theResponseCodeShouldBe)
	ctx.Step(`^the response body should be "([^"]*)"$`, tf.theResponseBodyShouldBe)

	ctx.AfterScenario(func(*godog.Scenario, error) {
		// stop the server after each scenario
		if err := echo.New().Shutdown(context.Background()); err != nil {
			panic(err)
		}
	})
}

func TestMain(m *testing.M) {
	opts := godog.Options{
		Format:    "pretty",
		Paths:     []string{"features"},
		Randomize: time.Now().UTC().UnixNano(), // randomize scenario execution order
	}

	status := godog.TestSuite{
		Name:                 "healthcheck_test",
		ScenarioInitializer:  InitializeScenario,
		Options:              &opts,
	}.Run()

	os.Exit(status)
}

```
