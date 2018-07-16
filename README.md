
[![Travis-CI Build
Status](https://travis-ci.org/hrbrmstr/mqtt.svg?branch=master)](https://travis-ci.org/hrbrmstr/mqtt)

# mqtt

Interoperate with ‘MQTT’ Message Brokers

## Description

‘MQTT’ is PPPP22 a machine-to-machine (‘M2M’)/“Internet of Things” connectivity
protocol. It was designed as an extremely lightweight publish/subscribe
messaging transport. It is useful for connections with remote locations
where a small code footprint is required and/or network bandwidth is at
a premium. For example, it has been used in sensors communicating to a
broker via satellite link, over occasional dial-up connections with
healthcare providers, and in a range of home automation and small device
scenarios. It is also ideal for mobile applications because of its small
size, low power usage, minimised data packets, and efficient
distribution of information to one or many receivers. Tools are provided
to interoperate with ‘MQTT’ message brokers in R.

## What You Need To Get This Working

You need to install the [`mosquitto`
libraries](https://mosquitto.org/download/) and ensure they’re on your
system `PATH` so R can find them. This will be easier in the future, but
coarse for the moment.

For macOS, that’s as easy as:

    brew install mosquitto

For Debian/Ubuntu, it will be something like:

    sudo apt install libmosquitto-dev libmosquittopp-dev

For Windows folks: install Linux or get a mac ;-) Seriously, I’m hoping
to have support for that soon.

## NOTICE

I had neglected to commit the code that makes the default client-id
unique. So if you had tried this just after the blog post and were
getting wonky results, that’s the cause. The default client id will
always be unique, now, but you should use your own, unique client ids in
production.

## Experimenting with a DSL workflow

I don’t have much need for these IoT queues yet but have been pondering
what an “R-like” way of working with them would be. Here’s one idea for
a domain-specific language workflow for watching many topics. We’ll use
an example of watching 3 different BBC subtitle topics simultaneously,
coloring each diffrently to tell one from the other.

``` r
library(mqtt)

sensor <- function(id, topic, payload, qos, retain, con) {
  if (topic == "bbc/subtitles/bbc_two_england/raw") {
    cat(crayon::cyan(topic), crayon::blue(readBin(payload, "character")), "\n", sep=" ")
  }
}

# NOTE: Use a unique name vs `hrbrunique`
mqtt_broker("hrbrnique", "test.mosquitto.org", 1883L) %>%
  mqtt_silence(c("error", "log")) %>% 
  mqtt_subscribe(
    "bbc/subtitles/bbc_one_london/raw", 
    function(id, topic, payload, qos, retain, con) { # regular anonymous function
      if (topic == "bbc/subtitles/bbc_one_london/raw")
        cat(crayon::yellow(topic), crayon::green(readBin(payload, "character")), "\n", sep=" ")
    }) %>%
  mqtt_subscribe("bbc/subtitles/bbc_news24/raw", ~{ # tilde shortcut function (passing in named, pre-known params)
    if (topic == "bbc/subtitles/bbc_news24/raw")
      cat(crayon::yellow(topic), crayon::red(readBin(payload, "character")), "\n", sep=" ")
  }) %>%
  mqtt_subscribe("bbc/subtitles/bbc_two_england/raw", sensor) %>% # named function
  mqtt_run() -> res
```

  - The only time anything is evaluated is when `mqtt_run()` is executed
    and it handles the main loop.
  - Provide connection info with `mqtt_broker()` & `mqtt_username_pw()`
    (WIP — certs TOTO).
  - Silence any default callback messages you want (WIP — need to add
    all events).
  - Subscribe to a topic, providing a handler function in one of three
    ways (each way requires you to use a specific signature for the
    function):
      - regular anonyous function (e.g. `fuction() {...}`)
      - `tidyeval`-esque tilde formula-function (e.g. `~{}`)
      - normal function reference (e.g. `fname`)

The signature must be:

``` r
function(id, topic, payload, qos, retain, con) {}
```

The first five are `mosquitto`-required params, but `con` provides
access to the lower-level object the DSL wraps, so you can, say, publish
something right after you receive a message (i.e. perform a calculation
or lookup and publish that as a response). For example, convert a
temperature value:

``` r
library(mqtt)

mqtt_broker("hrbrnique", "test.mosquitto.org", 1883L) %>%
  mqtt_silence(c("error", "log", "publish")) %>% 
  mqtt_subscribe("random/temperature", ~{
    if (topic == "random/temperature") {
      payload <- readBin(payload, "character")
      cat(crayon::cyan(topic), crayon::white(payload), "\n", sep=" ")
      resp <- weathermetrics::celsius.to.fahrenheit(as.numeric(payload))
      resp <- as.character(resp)
      con$publish_chr(0, "hrbrmstr/pub/tof", resp, 0, FALSE)
    }
  }) %>% 
  mqtt_run() -> res
```

Lots more to do / ponder before this is even really ready for 0.1.0
status.

## Simplest / Original Functionality

You can subscribe to a topic on a server over plaintext. No
authentication methods are supported (yet) and no ability to use
encryption exists (yet).

When you subscribe, you should pass in a “callback handler”. Said
handler should have the same “signature” as the built-in default one,
which
    is:

    mqtt_default_message_callback <- function(id, topic, payload, qos, retain)

Those parameters are:

  - `id`: the message id
  - `topic`: the message topic
  - `payload`: the message payload (raw)
  - `qos`: the effective qos for the message
  - `retain`: is this message marked as “retain”?

`payload` is a raw vector. The example below shows how to work with this
data.

If you return “`quit`” from this function, the subscription will be
closed and control return to R.

I can see that publishing data to an MQTT broker would be useful for R
so that is on the TODO.

PLEASE file an issue and/or PR if you have ideas in mind for this. I
have more “evil” notions in mind (quantifiying expousure of sensitive
data on public MQTT channels). I suspect others have “real” needs for
this.

## What’s Inside The Tin

The following functions are implemented:

### DSL

  - `mqtt_broker`: Provide broker connection setup information
  - `mqtt_username_pw`: Set username & passwords for the connection
  - `mqtt_silence`: Silence log and/or error or more callbacks
  - `mqtt_subscribe`: Subscribe to a channel identifying a callback
  - `mqtt_run`: Run an MQTT event loop
  - `mqtt_begin`: Initiate an MQTT connection
  - `mqtt_end`: Close an MQTT connection
  - `mqtt_loop`: Run an mqtt loop iteration

### Non-DSL

  - `mqtt_default_connection_callback`: mqtt default connection callback
    function
  - `mqtt_default_disconnection_callback`: mqtt default disconnection
    callback function (does - `nothing`)
  - `mqtt_default_message_callback`: mqtt default message callback
    function
  - `mqtt_silent__callback`: mqtt silent callback function (does
    nothing)
  - `topic_subscribe`: Subscribe to an MQTT Topic

## Installation

``` r
devtools::install_github("hrbrmstr/mqtt")
```

## Usage

``` r
library(mqtt)

# current verison
packageVersion("mqtt")
```

    ## [1] '0.2.0'

``` r
# internal function to see which mosquitto library is being used
print(mqtt:::mqtt_version())
```

    ## [1] "1.4.14"

### Live subtitles

For whatever reason, someone is using the public `test.mosquitto.org`
plaintext broker to push out live subtitles. It’s strangely mezmerizing
to watch it slowly scroll by. Let’s see the next 50 (as of the time this
Rmd
ran):

``` r
# We are going to cap it at 50 so we have to initialize a global we'll update
x <- 0

# Now, we need a callback function. This will get called everytime we get a message.
# the `topic` string will be passed in so you can compare that quickly.
# the `payload` is a raw vector since this can be pretty strange data (esp if you 
# subscribe to a wildcard).
# 
# You can use `rawToChar()` if you know it's going to be safe, but `readBin()` is
# a tad safter. Ideally, the package will have some sanitizing functions to make
# this easier and more robust.
my_msg_cb <- function(id, topic, payload, qos, retain) {
  
  if (topic == "bbc/subtitles/bbc_news24/compacted") { # when we see BBC msgs, we'll cat them
    x <<- x + 1
    cat(readBin(payload, "character"), "\n", sep="")
  } else {
    print(topic)
  }

  return(if (x==50) "quit" else "continue") # "continue" can be "". anything but "quit"
}

# now, we'll subscribe to a topic at `test.mosquitto.org` on port 1883. 
# those are defaults in `topic_subscribe()` to make it easier to have some quick
# fun with the package.
topic_subscribe(topic="bbc/subtitles/bbc_news24/compacted", message_callback=my_msg_cb)
```

    ## Default connect callback result: 0

    ##  to unite the ANC and lead it to
    ##  victory in elections of 2019, there
    ##  is a huge task to persuade the
    ##  majority of South African voters the
    ##  ANC has turned its back on the age
    ##  of corruption, of what is called
    ##  state capture where cronies of
    ##  President Zuma allegedly in return
    ##  for kickbacks were given chunks of
    ##  state enterprise, vast sums of
    ##  taxpayers' money. It is a big job to
    ##  persuade South Africans. The major
    ##  concern, it will define whether the
    ##  ANC continues to be the dominant
    ##  electoral force of this country. We
    ##  should have a result by around 5am,
    ##  as dawn comes up in Johannesburg.
    ##  Reporting on the election of a new
    ##  leader of the ANC and we expect the
    ##  outcome of the postal ballot, secret
    ##  ballot, many which have taken place
    ##  by
    ##  post
    ##  tomorrow.
    ##  MPs have expressed "serious doubts"
    ##  that the Ministry of Defence will be
    ##  able to afford all the new military
    ##  equipment it plans to buy.
    ##  A report by the Commons Defence
    ##  Select Committee says the MoD
    ##  will struggle to make the necessary
    ##  savings it needs to pay
    ##  for newjets, warships
    ##  and armoured vehicles,
    ##  as Ian Palmer reports.
    ##  She is the flagship
    ##  of the Royal Navy.
    ##  HMS Queen Elizabeth,
    ##  commissioned by Her Majesty the
    ##  Queen, early this month.
    ##  At 280 metres long,
    ##  she has space for 40
    ##  jet planes.
    ##  But defence in the 21st-century
    ##  does not come cheap.
    ##  The biggest warship the British Navy
    ##  has ever had cost more than £3
    ##  billion.
    ##  Another aircraft carrier
    ##  is being built in Scotland.

## Code of Conduct

Please note that this project is released with a [Contributor Code of
Conduct](CONDUCT.md). By participating in this project you agree to
abide by its terms.
