# I wrote a program to learn Goroutines

I learnt Go because I love how it packages everything nicely into small binary files and doing a multi stage docker build actually does something effective. Also, it includes a http package by default.

I was fascinated by download accelerators, the tool which downloads files in parts concurrently then stiches them together. To make sense of goroutines better, I decided to write one.

You can find the source code [here](https://github.com/snowset/ddacc-cli/).

I had certain requirements:
- Stop downloads if the internet is cut off or the server doesn't respond
- Graceful shutdown

As it is going to be a CLI app, we need flags.
```go
\\ main.go
flag.StringVar(&url, "url", "", "URL of the file")
flag.StringVar(&dir, "dir", os.Getenv("HOME")+"/Downloads/", "Directory where you want to download the file(s)")
flag.Int64Var(&connections, "connections", 4, "Number of concurrent connections")
flag.BoolVar(&withLog, "log", false, "show logs")
flag.StringVar(&urllist, "multiple", "",
	"Put filename of the .txt file containing the URL of the files you want to download\nURL 1\nURL 2\nURL 3\nURL n")
```

This function creates a download instance for the file. It requests for headers and then breaks the content length into segments by bytes. 
```go
// download.go
func NewDownloadInstance(url string, outputdir string, numParts int64, closeChan *chan os.Signal) (*Download, error) {
	resp, err := http.Head(url)
	if err != nil {
		return nil, err
	}

	defer resp.Body.Close()

	segmentSize := resp.ContentLength / numParts
	segments := make([][2]int64, numParts)

	for idx := range segments {
		start := int64(idx) * segmentSize
		end := start + segmentSize - 1

		if idx == int(numParts-1) {
			end = resp.ContentLength - 1
		}

		segments[idx] = [2]int64{start, end}
	}

	return &Download{
		URL:       url,
		OutputDir: outputdir,
		NumParts:  numParts,
		Client:    &http.Client{},
		Parts:     segments,
		closeChan: closeChan,
	}, nil
}
```

The Download function, which starts the download, creates the file. It has two channels, `errChan` for errors and `doneChan` to send message that download has been finished. It launches goroutines by parts and waits till all the goroutines finish downloading. After all the parts have been downloaded and written to the target file, it closes the channels and checks if there are any errors in the error channel, if exists, returns the first error. (As unbuffered channels work on LIFO)

```go
func (dm *Download) Download() error {
	// ----- create file, create channels errChan and doneChan and waitgroups-----
	for idx, segment := range dm.Parts {
		go func(idx int, segment [2]int64) {
			req, err := http.NewRequest("GET", dm.URL, nil)
			if err != nil {
				errChan <- err
				doneChan <- true
				return
			}

			resp, err := dm.Client.Do(req)
			if err != nil {
				// err to errChan and done
				return
			}

            // create a 1 MB buf (buffer for writing data)
			for {
				select {
				case <-doneChan:
					return
				case <-*dm.closeChan:
					// send interrupt err to errchan and done
					return
				default:
                    // Monitors the read function for time it takes to read 
					n, err := readWithTimeout(resp, buf, 60*time.Second) 
					if err != nil && err != io.EOF {
						// err to errChan and done
						return
					}
					if n == 0 {
						return
					}
					_, err = out.WriteAt(buf[:n], segment[0])
					if err != nil {
						// err to errChan and done
						return
					}
					segment[0] += int64(n)
				}
			}
            // waitgroup done
		}(idx, segment)
	}

	// close channels
	return <-errChan
}
```

To implement graceful shutdown, I added a goroutine to main function, which waits for SIGKILL or SIGINT and kills the active goroutine with error message, which deletes the files that haven't been properly downloaded.

```go
closeChan := make(chan os.Signal, 1)
signal.Notify(closeChan, syscall.SIGINT, syscall.SIGTERM)

go func() {
	<-closeChan
	close(closeChan)
}()
```