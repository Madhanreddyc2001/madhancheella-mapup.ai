package main

import (
	"encoding/json"
	"net/http"
	"sort"
	"sync"
	"time"
)

type RequestPayload struct {
	ToSort [][]int `json:"to_sort"`
}

type ResponsePayload struct {
	SortedArrays [][]int `json:"sorted_arrays"`
	TimeNs       int64   `json:"time_ns"`
}

func processSingle(w http.ResponseWriter, r *http.Request) {
	var reqPayload RequestPayload
	if err := json.NewDecoder(r.Body).Decode(&reqPayload); err != nil {
		http.Error(w, "Invalid JSON payload", http.StatusBadRequest)
		return
	}

	startTime := time.Now()

	var sortedArrays [][]int
	for _, subArray := range reqPayload.ToSort {
		sortedSubArray := make([]int, len(subArray))
		copy(sortedSubArray, subArray)
		sort.Ints(sortedSubArray)
		sortedArrays = append(sortedArrays, sortedSubArray)
	}

	response := ResponsePayload{
		SortedArrays: sortedArrays,
		TimeNs:       time.Since(startTime).Nanoseconds(),
	}

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(response)
}

func processConcurrent(w http.ResponseWriter, r *http.Request) {
	var reqPayload RequestPayload
	if err := json.NewDecoder(r.Body).Decode(&reqPayload); err != nil {
		http.Error(w, "Invalid JSON payload", http.StatusBadRequest)
		return
	}

	startTime := time.Now()

	var sortedArrays [][]int
	var wg sync.WaitGroup
	var mu sync.Mutex

	for _, subArray := range reqPayload.ToSort {
		wg.Add(1)
		go func(arr []int) {
			defer wg.Done()
			sortedSubArray := make([]int, len(arr))
			copy(sortedSubArray, arr)
			sort.Ints(sortedSubArray)

			mu.Lock()
			sortedArrays = append(sortedArrays, sortedSubArray)
			mu.Unlock()
		}(subArray)
	}

	wg.Wait()

	response := ResponsePayload{
		SortedArrays: sortedArrays,
		TimeNs:       time.Since(startTime).Nanoseconds(),
	}

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(response)
}

func main() {
	http.HandleFunc("/process-single", processSingle)
	http.HandleFunc("/process-concurrent", processConcurrent)

	port := "8000"
	serverAddr := ":" + port

	http.ListenAndServe(serverAddr, nil)
}
