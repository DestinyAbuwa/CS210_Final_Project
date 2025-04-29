# CS210 Final Project
# Destiny Abuwa

```
#include <iostream>
#include <fstream>
#include <sstream>
#include <string>
#include <unordered_map>
#include <list>
#include <vector>
#include <memory>
#include <random>     // For random replacement
#include <limits>     // For frequency comparisons

using namespace std;

// ------------------- CityKey Struct and Hash --------------------

// A structure to hold the country code and city name as the key to cache city populations
struct CityKey {
    string countryCode;
    string cityName;

    // Overload equality operator to compare CityKey objects
    bool operator==(const CityKey& other) const {
        return cityName == other.cityName && countryCode == other.countryCode;
    }
};

// Hash function for CityKey, required for using CityKey as a key in unordered_map
struct CityKeyHash {
    size_t operator()(const CityKey& key) const {
        return hash<string>()(key.cityName) ^ hash<string>()(key.countryCode); // Combine hashes of cityName and countryCode
    }
};

// ------------------- Base Cache Interface --------------------

// Abstract base class for city population caches
class CityCache {
public:
    // Virtual methods to be overridden in derived classes (get and put data into the cache)
    virtual bool get(const CityKey& key, string& population) = 0;
    virtual void put(const CityKey& key, const string& population) = 0;
    virtual ~CityCache() {}
};


// ------------------- LFU Cache --------------------

// Least Frequently Used (LFU) cache implementation
class LFUCache : public CityCache {
private:
    // Internal structure to represent each cache entry
    struct CacheEntry {
        string population;
        int frequency; // Track how often a city is accessed (frequency count)
        list<CityKey>::iterator orderIt; // Iterator to maintain insertion order
    };

    unordered_map<CityKey, CacheEntry, CityKeyHash> cacheMap; // Map to store city data
    list<CityKey> orderList; // List to store city access order (used for frequency eviction)
    const size_t maxSize = 10; // Maximum cache size

public:
    // Get city population from cache, updating frequency if found
    bool get(const CityKey& key, string& population) override {
        auto it = cacheMap.find(key);
        if (it != cacheMap.end()) {
            it->second.frequency++; // Increment frequency when accessed
            population = it->second.population;
            return true;
        }
        return false; // Return false if city is not in the cache
    }

    // Put city data into the cache
    void put(const CityKey& key, const string& population) override {
        auto it = cacheMap.find(key);
        if (it != cacheMap.end()) {
            it->second.population = population; // Update population if city is already in cache
            it->second.frequency++; // Increment frequency
            return;
        }

        // If the cache is full, evict the least frequently used city
        if (cacheMap.size() >= maxSize) {
            int minFreq = numeric_limits<int>::max();
            CityKey toRemove;

            // Find city with the minimum frequency for eviction
            for (const auto& k : orderList) {
                int freq = cacheMap[k].frequency;
                if (freq < minFreq) {
                    minFreq = freq;
                    toRemove = k;
                }
            }

            // Remove the least frequently used city from cache
            orderList.remove(toRemove);
            cacheMap.erase(toRemove);
        }

        // Add the new city to the cache and update frequency
        orderList.push_back(key);
        cacheMap[key] = {population, 1, --orderList.end()}; // Frequency starts at 1
    }
};

// ------------------- FIFO Cache --------------------

// First-In, First-Out (FIFO) cache implementation
class FIFOCache : public CityCache {
private:
    list<pair<CityKey, string>> cacheList; // List to maintain cache order (FIFO)
    unordered_map<CityKey, list<pair<CityKey, string>>::iterator, CityKeyHash> cacheMap; // Map for quick lookup
    const size_t maxSize = 10; // Maximum cache size

public:
    // Get city population from cache
    bool get(const CityKey& key, string& population) override {
        auto it = cacheMap.find(key);
        if (it != cacheMap.end()) {
            population = it->second->second;
            return true;
        }
        return false; // Return false if city is not in the cache
    }

    // Put city data into the cache
    void put(const CityKey& key, const string& population) override {
        auto it = cacheMap.find(key);
        if (it != cacheMap.end()) {
            cacheList.erase(it->second); // Remove the city from the list if it exists
            cacheMap.erase(it); // Remove from the map
        }

        // If the cache is full, evict the oldest city (FIFO)
        if (cacheList.size() >= maxSize) {
            auto last = cacheList.front(); // FIFO, so remove the first city
            cacheMap.erase(last.first);
            cacheList.pop_front();
        }

        // Add the new city to the cache
        cacheList.push_back({key, population});
        cacheMap[key] = --cacheList.end(); // Update the map with the new iterator
    }
};
