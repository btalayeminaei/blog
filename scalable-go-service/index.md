#  Writing Scalable Go Services

## Problem

The recent changes in our hotel vendors' APIs gave us an opportunity to evaluate and improve the backend services that interact with these APIs.
Specifically, we evaluated our rate loading strategies and looked for ways to improve load times as we migrate to the new APIs.
To assure customers are getting the best hotel deals at Upside, we must show that we have an inventory as complete as any other provider.
To accomplish this, we need to load a lot of rates from many different vendors.
We also need to get the complete inventory before the customer loses patience and leaves the site.

As we evaluated different strategies and planned our migration, an interesting nuance was that we were not always in control of the system bottlenecks.
We could only load rates as fast as our vendors provided them.
Although some allowed us to ask for multiple rates from multiple properties in a single request, others required us to make individual requests per property.
And even though these individual requests resolved quickly, the time complexity grew linearly with the number of hotels we needed to search.
We not only had to consider our overall rate loading processes, but also our vendors' limitations, and look for ways to fully utilize their resources.

## Concurrency and Go

Since we have no influence on how our vendors' APIs are designed, it was clear we had to parallelize our rate requests for individual hotels.
`go` is a relatively new language and is designed based on modern understanding of concurrency and parallel computing.
Its primitive types make designing and building concurrent applications simple and fun.
A `goroutine` can be thought of as an inexpensive and lightweight thread.
A `channel` enables two or more routines to communicate with each other, similar to a pipe.
The language also provides a multiway concurrent control switch through the `select` statement, which enables interactions with multiple `channels`.

Concurrency in `go` is based on [Tony Hoare's CSP](https://www.cs.cmu.edu/~crary/819-f09/Hoare78.pdf), which specifies the patterns of interactions between concurrent processes.
CSP takes a message passing approach to ensure data integrity.
Independent components should run concurrently and communicate with other components, if need be.
Concurrency is the composition of independently executing processes and their interactions.
Our goal should be to design a program that will run correctly and as efficiently as possible, regardless of the underlying resources available.

>[Do not communicate by sharing memory; instead, share memory by communicating.](<https://blog.golang.org/share-memory-by-communicating>)

## First Attempt

Following our agile mindset, we jumped into writing `go` and got to a MVP quickly.
We started by creating a solution that would simply use the concurrent primitives in `go` to parallelize loading rates for each hotel.
This new service would receive a list of hotels and spawn a new `goroutine` to fetch rates for each hotel.
We used a buffered channel to aggregate the results from each process.

```go
func getRates(hotelIds []int, checkInDate string, checkOutDate string) []hotelWithRates {
  var rates []hotelWithRates
  ch := make(chan hotelWithRates, len(hotelIds))

  for _, hotelID := range hotelIds {
    go getHotelRates(ch, hotelID, checkInDate, checkOutDate)
  }

  for range hotelIds {
    hotelWithRates := <-ch
    rates = append(rates, hotelWithRates)
  }

  if rates == nil || len(rates) < 1 {
    return rates, fmt.Errorf("no rates found for search criteria")
  }

  return rates, nil
}

```

We were quickly able to implement this solution and see an improvement in the load time, which was a great proof of concept.
However, this implementation allowed the incoming requests to manipulate resource usage and the number of calls to our external vendors.
Although `goroutines` are cheap and a typical `go` program can spawn thousands of them during its runtime, any single request would spawn hundreds of routines.
Even if the request was validated to ensure the underlying capacities were not abused, this solution did not give us the freedom to configure how much resources we wanted to make available.


## The Worker Model

The proof of concept gave us confidence that we were on the right path, so we decided to take the time to design a more robust concurrent model to address some of our concerns.
We began by identifying all the components that could conceptually run independent of the others in the system.
The white board gave us the freedom to compose these independently executing modules and find areas where we needed to add communication channels between them.


<p align="center">
<img src="./pictures/second.jpg" width="600" heigth:"400"/>
</p>

The route handler, the token manager, and the rate loader were the three components we identified as necessary for processing a request to load rates.
It was not immediately obvious how these components would run independently.
The rate loader takes a validated request from the route handler and needs a token before contacting the vendor.
However, we noticed that multiple rate loaders can execute independently once they have all of the necessary information.
Once we facilitate proper communication between the components, we can start viewing them as workers that receive required parameters from `channels`, perform their task, and communicate their result through `channels`.
Groups of workers that perform the same tasks can be regarded as a pool of workers.
They all listen on a single channel for requests they can process.


<p align="center">
<img src="./pictures/third.jpg" width="600" heigth:"400"/>
</p>


How many rate loaders?
It depends on the application, but this is now a configurable parameter.
It does not have to be an exact number either.
One might configure a lower and an upper bound for how many rate loaders they want in their app and can add logic to dynamically scale to throw limits based on the current load.

## Sample Code

#### Main

```go
func main() {
  var wg sync.WaitGroup
  rateLoaderCount := 10
  tokenRQChan := make(chan domain.TokenRQ)
  ratesRQChan := make(chan domain.RateRQ, rateLoaderCount*2)

  var tokenLoader = TokenLoader{
    In:  tokenRQChan,
  }
  var rateLoaderPool = RateLoaderPool{
    Count:       rateLoaderCount,
    In:          ratesRQChan,
    TokenRQChan: tokenRQChan,
  }

  appRouter := mux.NewRouter()
  appRouter.HandleFunc("/rates", RatesHandler(ratesRQChan))
  srv := http.Server{
    Addr:    "127.0.0.1:8080",
    Handler: appRouter,
  }

  wg.Add(1)
  go func() {
    tokenLoader.Start()
    wg.Done()
  }()

  wg.Add(1)
  go func() {
    rateLoaderPool.Start()
    wg.Done()
  }()

  wg.Wait()

  // omitting implementation of gracefully shutting down all the workers
}
```

#### Rates Route Handler

```go
func RatesHandler(ratesRQChan chan domain.RateRQ) func(w http.ResponseWriter, r *http.Request) {
  return func(w http.ResponseWriter, r *http.Request) {
    var rates []types.HotelRateInfo
    rateRSChan := make(chan RateRS, len(r.HotelIds))
    for _, hotelID := range r.HotelIds {
      ratesRQChan <- RateRQ{
        HotelID:      hotelID,
        CheckInDate:  r.CheckInDate,
        CheckOutDate: r.CheckOutDate,
        RateRSChan:   rateRSChan,
      }
    }
    for range p.HotelIds {
      res := <-rateRSChan
      rates = append(rates, res.HotelRateInfo)
    }
    w.WriteHeader(http.StatusOK)
    w.Write(json.Marshal(rates))
  }
}
```

#### Rate Loader Pool

```go
type RateLoaderPool struct {
  Count       int
  In          chan RateRQ
  TokenRQChan chan<- TokenRQ
  stop        chan struct{}
}

func (rlp *RateLoaderPool) Start() {
  rlp.stop = make(chan struct{}, 1)
  var wg sync.WaitGroup
  for i := 0; i < rlp.Count; i++ {
    wg.Add(1)
    tokenRSChan := make(chan TokenRS)
    go func() {
    loop:
      for {
        select {
          case rq := <-rlp.In:
            rlp.TokenRQChan <- TokenRQ{...}
            token := <-tokenRSChan
            r := getRateFromVendor(...)
            rq.RateRSChan <- RateRS{r}
          case <-rlp.stop:
            break loop
        }
      }
      wg.Done()
    }()
  }
  wg.Wait()
}

func (rlp *RateLoaderPool) Stop() {
  close(rlp.stop)
}
```

#### Token Loader

```go
type TokenLoader struct {
  In   <-chan TokenRQ
  stop chan struct{}
}

func (tl *TokenLoader) Start() {
  tl.stop = make(chan struct{}, 1)
  var token *TokenRS
loop:
  for {
    select {
      case rq := <-tl.In:
        rq.TokenRSChan <- getAuthenticationToken(...)
      case <-tl.stop:
        break loop
    }
  }
}

func (tl *TokenLoader) Stop() {
  close(tl.stop)
}
```

## Conclusion
Changes to our vendors' APIs forced us to evaluate and improve our strategy for loading hotel rates.
Rather than porting over the existing code and making it compatible with the new APIs, we decided to invest the time and found a modern tool and approach to solving the problem with `go`.
`go`'s concurrency primitives made it easy to parallelize rate requests and see the impact in our overall load time.
A deeper look into how `go` concurrency was designed and using the ideas of CSP allowed us to build a scalable and configurable solution with confidence.


## Helpful Links
- <https://medium.com/@1_00794/parallelism-models-actors-vs-csp-vs-multithreading-f1bab1a2ed6b>
- <https://www.youtube.com/watch?v=cN_DpYBzKso>
- <http://www.usingcsp.com/cspbook.pdf>

