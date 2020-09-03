# Changes to s3tester in this repo

Data generated is random to prevent extreme DRR in SUT.

New arguments:

	-fanout - Fanout support for GET/PUT operations (remote copies in SUT)
	-interDelay - Adds a delay between requests, measured in milliseconds. For example a value of 200 means a PUT every 200ms.
	-opThreshold - Threshold in ms for operations (outputs hit/miss %). For example a value of 100 will count all requests that elapsed more than 100ms.

# s3tester - S3 Performance Benchmarking 

The goal of s3tester is to be a lightweight, S3 performance testing utility. It is solely focused on S3 testing.

This tool is in active development - please submit feature requests in the issues page

## s3tester latest version


	2.1.0

# Installation

	$ go get github.com/s3tester/s3tester

## Minimum Requirements
	
	Go 1.7 or higher

# Usage

## Setting your s3 credentials

There are multiple options for setting credentials.

Environment Variables:

    $ export AWS_ACCESS_KEY_ID=AKIAINZFCN46TISVUUCA
    $ export AWS_SECRET_ACCESS_KEY=VInXxOfGtEIwVck4AdtUDavmJf/qt3jaJEAvSKZO

AWS credential file: see the --profile option below for details.

## Command line options

Usage of ./s3tester:

    -bucket string
        bucket name (needs to exist) (default "test")
    -concurrency int
        Maximum concurrent requests (0=scan concurrency, run with ulimit -n 16384) (default 1)
    -consistency string
        The StorageGRID consistency control to use for all requests. Does nothing against non StorageGRID systems. (all, available, strong-global, strong-site, read-after-new-write, weak)
    -cpuprofile string
        write cpu profile to file
    -days int
        The number of days that the restored object will be available for (default 1)
    -duration value
        Test duration in seconds
    -endpoint string
        target endpoint(s). If multiple endpoints are specified separate them with a ','. Note: the concurrency must be a multiple of the number of endpoints. (default "https://127.0.0.1:18082")
    -json
        The result will be printed out in JSON format if this flag exists
    -lockstep
        Force all threads to advance at the same rate rather than run independently
    -logdetail string
        write detailed log to file
    -loglatency string
        write latency histogram to file
    -metadata string
        The metadata to use for the objects. The string must be formatted as such: 'key1=value1&key2=value2'. Used for put, updatemeta, multipartput, putget and putget9010r.
    -no-sign-request
        Do not sign requests. Credentials will not be loaded if this argument is provided.
    -operation string
        operation type: put, multipartput, get, puttagging, updatemeta, randget, delete, options, head, restore (default "put")
    -overwrite int
        Turns a PUT/GET/HEAD into an operation on the same s3 key. (1=all writes/reads are to same object, 2=threads clobber each other but each write/read is to unique objects).
    -partsize int
        Size of each part (min 5MiB); only has an effect when a multipart put is used (default 5242880)
    -prefix string
        object name prefix (default "testobject")
    -profile string
        Use a specific profile from AWS CLI credential file (https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-profiles.html).
    -range string
        Specify range header for GET requests
    -ratelimit float
        the total number of operations per second across all threads (default 1.7976931348623157e+308)
    -region string
        Region to send requests to (default "us-east-1")
    -repeat int
        Repeat each S3 operation this many times, by default doesn't repeat (i.e. repeat=0)
    -requests value
        Total number of requests (default 1000)
    -retries int
        Number of retry attempts. Default is 0.
    -retrysleep int
        How long to sleep in between each retry in milliseconds. Default (0) is to use the default retry method which is an exponential backoff.
    -rr
        Reduced redundancy storage for PUT requests
    -size int
        Object size. Note that s3tester is not ideal for very large objects as the entire body must be read for v4 signing and the aws sdk does not support v4 chunked. Performance may degrade as size increases due to the use of v4 signing without chunked support (default 30720)
    -tagging string
        The tag-set for the object. The tag-set must be formatted as such: 'tag1=value1&tage2=value2'. Used for put, puttagging, putget and putget9010r.
    -tier string
        The retrieval option for restoring an object. One of expedited, standard, or bulk. AWS default option is standard if not specified (default "standard")
    -uniformDist string
        Generates a uniform distribution of object sizes given a min-max size (10-20)
    -verify int
        Verify the retrieved data on a get operation - (0=disable verify(default), 1=normal put data, 2=multipart put data). If verify=2, partsize is required and default partsize is set to 5242880.
    -workload string
        Filepath to JSON either a replay file generated by the auditAnalysis tool to play an exact workload on a grid or a Mixedworkload json file which allows a user to specify a mixture of operations. A sample mixed workload file must be in the format
        '{'mixedWorkload':
        [{'operation':'put','ratio':25},
        {'operationType':'get','ratio':25},
        {'operationType':'updatemeta','ratio':25},
        {'operationType':'delete','ratio':25}]}'.  
        NOTE: The order of operations specified will generate the requests in the same order.
        I.E. If you have delete followed by a put, but no objects on your grid to delete, all your deletes will fail.

## Exit code
`1` One or more requests has failed.

# Examples

## Writing objects into a bucket

    ./s3tester -concurrency=128 -size=20000000 -operation=put -requests=20000 -endpoint="10.96.105.5:8082" -prefix=3

- Starts writing objects into the default bucket "test".
- The bucket needs to be created prior to running s3tester.
- The naming of the ingested objects will be `3-object#` where "3" is the prefix specified and `object#` is a sequential number starting from zero and going to the number of requests.
- This command will perform a total of 20,000 PUT requests (or in this case slightly less because 20,000 does not divide by 128).
- The object size is 20,000,000 bytes.
- Replace the sample IP/port combination with the one you are using.

## Reading objects from a bucket (and other operations)
    ./s3tester -concurrency=128 -operation=get -requests=200000 -endpoint="10.96.105.5:8082" -prefix=3

- Matches the request above and will read the same objects written in the same sequence.
- If you use the `randget` operation the objects will be read in random order simulating a random-access workload.
- If you use the `head` operation then the S3 HEAD operation will be performed against the objects in sequence.
- If you use the `delete` operation then the objects will be deleted.

As of version 2.1.0 the concurrency on a retrieval operation can be different from the concurrency used to ingest the objects. The goal is to save time by ingesting data once and retrieving at different concurrencies
to observe the impact on performance. However, the number of requests has to match the number that was actually ingested. For example, if we ingest with concurrency 1000 and requests set to 1100 then only 1000 requests
will actually be ingested (1100 - 1100%1000) to keep the number of requests per client thread equal. Now when performing the retrieval the number of requests specified must be 1000, not 1100.

# Interpreting the results
	        --- Total Results ---
	Operation: put
	Concurrency: 64
	Total number of requests: 99968
	Total number of unique objects: 99968
	Failed requests: 0
	Total elapsed time: 2m43.251246249s
	Average request time: 101.057175ms
	Minimum request time: 13.84ms
	Maximum request time: 712.75ms
	Nominal requests/s: 633.3
	Actual requests/s: 612.4
	Content throughput: 2.392018 MB/s
	Average Object Size: 4096
	Response Time Percentiles
	50     :   93.91 ms
	75     :   114.68 ms
	90     :   140.4 ms
	95     :   166 ms
	99     :   331.71 ms
	99.9   :   492.57 ms
	Latency(ms) : Operations
	  0 - 1   : 0     |
	  2 - 3   : 0     |
	  4 - 7   : 0     |
	  8 - 15  : 7     |
	 16 - 31  : 945   ||
	 32 - 63  : 12093 ||||||||||||||
	 64 - 127 : 71662 |||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||
	128 - 255 : 13671 ||||||||||||||||
	256 - 511 : 1505  ||
	512 - 713 : 85    |

- `Nominal requests/s` is calculated ignoring any client side overheads.  This number will always be higher than actual requests/s.  If those two numbers diverge significantly it can be an indication that the client machine isn't capable of generating the required workload and you may want to consider using multiple machines.
- `Actual requests/s` is the total number of requests divided by the total elapsed time in seconds.
- `Content throughtput` is the total amount of data ingested and retrieved in MB divided by the total elapsed time in seconds.
- `Total number of unique objects` is the total number of unique objects being operated on successfully.

For per request details, s3tester can be run with the `-logdetail` option for capturing all the request latencies into a `.csv` file.
