---
layout: post
title: "Designing Parking Structure"
date: 2015-05-04 21:20
comments: true
categories:
 - Object Oriented Programming
 - C++

tags:
- '2015'
---


## Designing Parking Structure

{% include toc %}

<br><br>
### Problem Statement

Design a parking lot using object-oriented principles.

To come up with an object-oriented design for Parking lot lets first go over the requirements of the system

- Parking lot has multiple floors
- Each floor have multiple parking slots
- Parking slots are for different vehicles like bike, car, and bus
- Parking slots are of different sizes based on vehicle
- Some parking slots are reserved for handicaps, Lets day 5 slots for each vehicle type on every floor is reserved for handicap
- Each parking floor have a parking meter
- How many total parking slots are available in the parking lot?
- How many total parking slots are available on each parking floor?

<br><br>
### Identify Components

Based on the above requirements following are the major components in the system

- Parking lot
- Parking floors
- Parking slots
    - Car
            - Regular
            - Handicap
    - Bike
    - Bus
- Parking Meter
    - Per Hour
    - Per Day
    - Per Month

<br><br>
### Identify Classes and Relationships

![center-aligned-image](/images/parkinglot.png)

<br><br>
### Parking Lot - Header

{% highlight c++ linenos %}
#ifndef parkinglot_hpp
#define parkinglot_hpp

#include <stdio.h>
#include <string>
#include <vector>
#include <iostream>
#endif /* parkinglot_hpp */

using namespace std;

enum SlotSize {compact = 1, large};
enum SlotType {handicap = 1, regular};
enum SlotRanges {handicapCompactStart = 0,
                handicapCompactEnd = 3,
                handicapLargeStart = handicapCompactEnd + 1,
                handicapLargeEnd = handicapLargeStart + 3,
                regularCompactStart = handicapLargeEnd + 1,
                regularCompactEnd = regularCompactStart + 25,
                regularLargeStart = regularCompactEnd + 1,
                regularLargeEnd = 50
};
const int PARKINGSLOTSCOUNT=50;

/*************************
 * Class vehical
 *************************/
class vehical
{
protected:
    string licensePlate;
    SlotSize parkingslotSize;
    SlotType vehicalSlotType;

public:
    vehical() {}
    vehical(string licensePlate, SlotSize parkingslotSize, SlotType vehicalType) {
        this->licensePlate = licensePlate;
        this->parkingslotSize = parkingslotSize;
        this->vehicalSlotType = vehicalType;
    }

    string getLicensePlate() {
        return licensePlate;
    }

    SlotSize getVehicalParkingSize() {
        return parkingslotSize;
    }

    SlotType getVehicalSlotType() {
        return vehicalSlotType;
    }
};

/*************************
 * Class bike
 *************************/
class bike:public vehical
{
public:
    bike() {}
    bike(string licensePlate, SlotSize parkingslotSize, SlotType vehicalType) {
        this->licensePlate = licensePlate;
        this->parkingslotSize = parkingslotSize;
        this->vehicalSlotType = vehicalType;
    }
};

/*************************
 * Class bus
 *************************/
class bus:public vehical
{
public:
    bus() {}
    bus(string licensePlate, SlotSize parkingslotSize, SlotType vehicalType) {
        this->licensePlate = licensePlate;
        this->parkingslotSize = parkingslotSize;
        this->vehicalSlotType = vehicalType;
    }
};

/*************************
 * Class car
 *************************/
class car:public vehical
{
public:
    car() {}
    car(string licensePlate, SlotSize parkingslotSize, SlotType vehicalType) {
        this->licensePlate = licensePlate;
        this->parkingslotSize = parkingslotSize;
        this->vehicalSlotType = vehicalType;
    }
};

/*************************
 * Class parkingslot
 *************************/
class parkingslot
{
private:
    int slotNumber;
    SlotSize slotSize;
    SlotType slotType;
    bool available;
    vehical* vehicalDetails;

public:
    parkingslot() {}
    parkingslot(int slotNumber, SlotSize slotSize, SlotType slotType) {
        this->slotNumber = slotNumber;
        this->slotSize = slotSize;
        this->slotType = slotType;
        available = true;
        vehicalDetails = NULL;
    }

    bool park(vehical* vehicalDetails);
    bool removeVehical();
    int getSlotNumber();
    SlotSize getSlotSize();
    SlotType getSlotType();
    bool isAvailable();
    vehical* getVehicalDetails();
};

/*************************
 * Class parkingfloor
 *************************/
class parkingfloor
{
private:
    int floorNumber;
    int totalParkingSlots;
    int totalAvailableSlots;
    vector<parkingslot*> slots;
    int handicapCompactTakenCount;
    int handicapLargeTakenCount;
    int regularCompactTakenCount;
    int regularLargeTakenCount;

public:
    parkingfloor() {}
    parkingfloor(int floorNumber, int totalParkingSlots) {
        this->floorNumber = floorNumber;
        this->totalParkingSlots = totalParkingSlots;

        for (int i = handicapCompactStart; i <= handicapCompactEnd; i++) {
            parkingslot* obj = new parkingslot(i, compact, handicap);
            slots.push_back(obj);
        }
        for (int i = handicapLargeStart; i <= handicapLargeEnd; i++) {
            parkingslot* obj = new parkingslot(i, large, handicap);
            slots.push_back(obj);
        }
        for (int i = regularCompactStart; i <= regularCompactEnd; i++) {
            parkingslot* obj = new parkingslot(i, compact, regular);
            slots.push_back(obj);
        }
        for (int i = regularLargeStart; i <= regularLargeEnd; i++) {
            parkingslot* obj = new parkingslot(i, large, regular);
            slots.push_back(obj);
        }
        handicapCompactTakenCount = 0;
        handicapLargeTakenCount = 0;
        regularCompactTakenCount = 0;
        regularLargeTakenCount = 0;
    }

    int getTotalAvailableSlots(SlotSize parkingSlotSize, SlotType parkingSlotType);
    int findandParkInNextAvailableSlot(vehical* vehicalDetails);
    void removeVehicalFromParkingSlot(int slotNumber);
};

/*************************
 * Class parkinglot
 *************************/
class parkinglot
{
private:
    vector<parkingfloor*> floors;

public:
    parkinglot() {}
    parkinglot(int totalFloors) {
        for (int i = 0; i < totalFloors; i++) {
            parkingfloor* obj = new parkingfloor(i, PARKINGSLOTSCOUNT);
            floors.push_back(obj);
        }
    }

    vector<parkingfloor*> getFloors() {
        return floors;
    }

    bool parkVehical(vehical* vehicalDetails);
    void printReport();
};

void parkinglotDriver();
{% endhighlight %}



### Parking Lot - Source

{% highlight c++ linenos %}
#include "parkinglot.hpp"

/******************************
 * parkingslot methods
 *****************************/
bool parkingslot::park(vehical* vehicalDetails)
{
    if (available == false ||
        (vehicalDetails->getVehicalParkingSize() != slotSize &&
         vehicalDetails->getVehicalSlotType() != slotType)) {
        return false;
    }

    available = false;
    this->vehicalDetails = vehicalDetails;

    return true;
}

bool parkingslot::removeVehical()
{
    this->vehicalDetails = NULL;
    this->available = true;

    return true;
}

int parkingslot::getSlotNumber()
{
    return slotNumber;
}

SlotSize parkingslot::getSlotSize()
{
    return slotSize;
}

SlotType parkingslot::getSlotType()
{
    return slotType;
}

bool parkingslot::isAvailable()
{
    return available;
}

vehical* parkingslot::getVehicalDetails()
{
    return vehicalDetails;
}


/******************************
 * parkingfloor methods
 *****************************/
int parkingfloor::getTotalAvailableSlots(SlotSize parkingSlotSize,
                    SlotType parkingSlotType)
{
    if (parkingSlotSize == compact && parkingSlotType == handicap) {
        return (handicapCompactEnd - handicapCompactStart) - handicapCompactTakenCount;
    } else if (parkingSlotSize == compact && parkingSlotType == regular) {
        return (regularCompactEnd - regularCompactStart) - regularCompactTakenCount;
    } else if (parkingSlotSize == large && parkingSlotType == handicap) {
        return (handicapLargeEnd - handicapLargeStart) - handicapLargeTakenCount;
    } else if (parkingSlotSize == large && parkingSlotType == regular) {
        return (regularLargeEnd - regularLargeStart) - regularLargeTakenCount;
    }

    return -1;
}

int parkingfloor::findandParkInNextAvailableSlot(vehical* vehicalDetails)
{
    int slotNumber = -1;

    // Get vehical parkingSlotSize and parkingSlotType
    if (vehicalDetails->getVehicalParkingSize() == compact) {
        if (vehicalDetails->getVehicalSlotType() == handicap) {
            // Find available slot for compact and handicap
            for (int i = handicapCompactStart; i <= handicapCompactEnd; i++) {
                if (slots[i]->isAvailable() && slots[i]->park(vehicalDetails)) {
                    handicapCompactTakenCount++;
                    slotNumber = i;
                    break;
                }
            }
        } else if (vehicalDetails->getVehicalSlotType() == regular) {
            // Find available slot for compact and regular
            for (int i = regularCompactStart; i <= regularCompactEnd; i++) {
                if (slots[i]->isAvailable() && slots[i]->park(vehicalDetails)) {
                    regularCompactTakenCount++;
                    slotNumber = i;
                    break;
                }
            }
        }
    } else if (vehicalDetails->getVehicalParkingSize() == large) {
        if (vehicalDetails->getVehicalSlotType() == handicap) {
            // Find available slot for large and handicap
            for (int i = handicapLargeStart; i <= handicapLargeEnd; i++) {
                if (slots[i]->isAvailable() && slots[i]->park(vehicalDetails)) {
                    handicapLargeTakenCount++;
                    slotNumber = i;
                    break;
                }
            }
        } else if (vehicalDetails->getVehicalSlotType() == regular) {
            // Find available slot for large and regular
            for (int i = regularLargeStart; i <= regularLargeEnd; i++) {
                if (slots[i]->isAvailable() && slots[i]->park(vehicalDetails)) {
                    regularLargeTakenCount++;
                    slotNumber = i;
                    break;
                }
            }
        }
    }

    return slotNumber;
}

void parkingfloor::removeVehicalFromParkingSlot(int slotNumber)
{
    if (slotNumber < 0 || slotNumber > PARKINGSLOTSCOUNT) {
        return ;
    }

    vehical* vehicalDetails = slots[slotNumber]->getVehicalDetails();
    // Get vehical parkingSlotSize and parkingSlotType
    if (vehicalDetails->getVehicalParkingSize() == compact) {
        if (vehicalDetails->getVehicalSlotType() == handicap &&
            handicapCompactTakenCount > 0) {
            handicapCompactTakenCount--;
        } else if (vehicalDetails->getVehicalSlotType() == regular &&
                   regularCompactTakenCount > 0) {
            regularCompactTakenCount--;
        }
    } else if (vehicalDetails->getVehicalParkingSize() == large) {
        if (vehicalDetails->getVehicalSlotType() == handicap &&
            handicapLargeTakenCount > 0) {
            handicapLargeTakenCount--;
        } else if (vehicalDetails->getVehicalSlotType() == regular &&
                   regularLargeTakenCount > 0) {
            regularLargeTakenCount--;
        }
    }

    slots[slotNumber]->removeVehical();
}

bool parkinglot::parkVehical(vehical* vehicalDetails)
{
    for (int i = 0; i < 3; i++) {
        int slotNumber = floors[i]->findandParkInNextAvailableSlot(vehicalDetails);
        if (slotNumber != -1) {
            cout << "Parked " << vehicalDetails->getLicensePlate() <<
         " on Floor '" << i << "' and slot '" << slotNumber << "'" << endl;

            return true;
        }
    }

    return false;
}

void parkinglot::printReport()
{
    for (int i = 0; i < floors.size(); i++) {
        cout << endl<< "***************************" << endl;
        cout << "Floor #" << i << endl;
        cout << "***************************" << endl;
        cout << "Slot range for compact, handicap [" << handicapCompactStart <<
        ", " << handicapCompactEnd << "]" << endl;
        cout << "Total available slots for compact, handicap = " <<
        floors[i]->getTotalAvailableSlots(compact, handicap) << endl;
        cout << "Slot range for large, handicap [" << handicapLargeStart <<
        ", " << handicapLargeEnd << "]" << endl;
        cout << "Total available slots for large, handicap = " <<
        floors[i]->getTotalAvailableSlots(large, handicap) << endl;
        cout << "Slot range for compact, regular [" << regularCompactStart <<
        ", " << regularCompactEnd << "]" << endl;
        cout << "Total available slots for compact, regular = " <<
        floors[i]->getTotalAvailableSlots(compact, regular) << endl;
        cout << "Slot range for large, regular [" << regularLargeStart <<
        ", " << regularLargeEnd << "]" << endl;
        cout << "Total available slots for large, regular = " <<
        floors[i]->getTotalAvailableSlots(large, regular) << endl;
    }

    return;
}

/******************************
 * parkinglot driver
 *****************************/
void parkinglotDriver()
{
    parkinglot parkingLotObj(3);
    vector<parkingfloor*> floors = parkingLotObj.getFloors();

    parkingLotObj.printReport();
    for (int i = 0; i < 30; i++) {
        string temp = "carCompactRegular" + std::to_string(i);
        car obj1(temp, compact, regular);
        parkingLotObj.parkVehical(&obj1);

        temp = "carCompactHandicap" + std::to_string(i);
        car obj2(temp, compact, handicap);
        parkingLotObj.parkVehical(&obj2);
    }

    parkingLotObj.printReport();

    return;
}
{% endhighlight %}

<br><br><br><br>
