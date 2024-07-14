#include <iostream>
#include <vector>
#include <map>
#include <string>
#include <algorithm>
#include <ctime>

using namespace std;

const int DAYS_IN_JULY = 31;
const int JULY_YEAR = 2024;
const int FIRST_DAY_OF_JULY = 1; // Assuming July 1st, 2024 is a Monday (0=Sunday, 1=Monday, ..., 6=Saturday)

struct Event {
    string title;
    int day;
    string startTime;
    string endTime;
    string repeating; // none, daily, weekly
};

class Calendar {
private:
    map<int, vector<Event>> events; // Maps each day to a list of events
    vector<int> dayOffs;

    bool isValidTimeFormat(const string& time) {
        if (time.length() != 4) return false;
        if (!isdigit(time[0]) || !isdigit(time[1]) || !isdigit(time[2]) || !isdigit(time[3])) return false;
        int hour = stoi(time.substr(0, 2));
        int minute = stoi(time.substr(2, 2));
        return (hour >= 0 && hour < 24) && (minute == 0 || minute == 30);
    }

    bool isOvernightEvent(const string& startTime, const string& endTime) {
        int startHour = stoi(startTime.substr(0, 2));
        int endHour = stoi(endTime.substr(0, 2));
        return (startHour >= 22 || endHour <= 6);
    }

    bool isTimeOverlap(const Event& newEvent, const Event& existingEvent) {
        return !(newEvent.endTime <= existingEvent.startTime || newEvent.startTime >= existingEvent.endTime);
    }

    bool isWeekend(int day) {
        int dayOfWeek = (FIRST_DAY_OF_JULY + (day - 1)) % 7;
        return dayOfWeek == 0 || dayOfWeek == 6; // Sunday or Saturday
    }

    bool isValidDate(int day) {
        return day >= 1 && day <= DAYS_IN_JULY;
    }

    bool isDateInPast(int day) {
        time_t t = time(0); 
        struct tm * now = localtime(&t);
        int currentYear = now->tm_year + 1900;
        int currentMonth = now->tm_mon + 1;
        int currentDay = now->tm_mday;

        if (currentYear > JULY_YEAR) return true;
        if (currentYear == JULY_YEAR && currentMonth > 7) return true;
        if (currentYear == JULY_YEAR && currentMonth == 7 && currentDay > day) return true;

        return false;
    }

    vector<string> generateTimeSlots() {
        vector<string> timeSlots;
        for (int hour = 0; hour < 24; ++hour) {
            for (int minute = 0; minute < 60; minute += 30) {
                string hh = (hour < 10 ? "0" : "") + to_string(hour);
                string mm = (minute < 10 ? "0" : "") + to_string(minute);
                timeSlots.push_back(hh + mm);
            }
        }
        return timeSlots;
    }

    string addMinutes(const string& time, int minutesToAdd) {
        int hour = stoi(time.substr(0, 2));
        int minute = stoi(time.substr(2, 2));
        minute += minutesToAdd;
        if (minute >= 60) {
            minute -= 60;
            hour += 1;
        }
        string hh = (hour < 10 ? "0" : "") + to_string(hour);
        string mm = (minute < 10 ? "0" : "") + to_string(minute);
        return hh + mm;
    }

public:
    void addDayOff(int day) {
        if (isValidDate(day) && find(dayOffs.begin(), dayOffs.end(), day) == dayOffs.end()) {
            dayOffs.push_back(day);
            cout << "Day " << day << " marked as day off.\n";
        } else {
            cout << "Invalid day or already marked as day off.\n";
        }
    }

    void scheduleEvent() {
        Event newEvent;
        cout << "Enter event title: ";
        cin.ignore();
        getline(cin, newEvent.title);

        cout << "Enter event date (day): ";
        cin >> newEvent.day;
        if (!isValidDate(newEvent.day) || isDateInPast(newEvent.day)) {
            cout << "Invalid date or date is in the past.\n";
            return;
        }

        cout << "Enter event start time (HHMM): ";
        cin >> newEvent.startTime;
        if (!isValidTimeFormat(newEvent.startTime)) {
            cout << "Invalid time format.\n";
            return;
        }

        cout << "Enter event end time (HHMM): ";
        cin >> newEvent.endTime;
        if (!isValidTimeFormat(newEvent.endTime) || newEvent.endTime <= newEvent.startTime) {
            cout << "Invalid end time format or end time is not after start time.\n";
            return;
        }

        if (isOvernightEvent(newEvent.startTime, newEvent.endTime)) {
            cout << "Overnight events are not allowed (between 22:00 and 06:00).\n";
            return;
        }

        cout << "Is this a repeating event? (none/daily/weekly): ";
        cin >> newEvent.repeating;

        // Validate no overlap and no day off or weekend without permission
        if (find(dayOffs.begin(), dayOffs.end(), newEvent.day) != dayOffs.end()) {
            cout << "Cannot schedule an event on a day off.\n";
            return;
        }

        if (isWeekend(newEvent.day)) {
            char permission;
            cout << "This is a weekend. Do you want to proceed? (y/n): ";
            cin >> permission;
            if (permission != 'y') {
                cout << "Event not scheduled on weekend.\n";
                return;
            }
        }

        for (const auto& event : events[newEvent.day]) {
            if (isTimeOverlap(newEvent, event)) {
                cout << "Event overlaps with an existing event.\n";
                return;
            }
        }

        events[newEvent.day].push_back(newEvent);
        cout << "Event scheduled successfully.\n";

        // Handle repeating events
        if (newEvent.repeating == "daily") {
            for (int i = newEvent.day + 1; i <= DAYS_IN_JULY; ++i) {
                if (find(dayOffs.begin(), dayOffs.end(), i) == dayOffs.end() && !isDateInPast(i)) {
                    events[i].push_back(newEvent);
                }
            }
        } else if (newEvent.repeating == "weekly") {
            for (int i = newEvent.day + 7; i <= DAYS_IN_JULY; i += 7) {
                if (find(dayOffs.begin(), dayOffs.end(), i) == dayOffs.end() && !isDateInPast(i)) {
                    events[i].push_back(newEvent);
                }
            }
        }
    }

    void viewEventsByDate(int day) {
        if (!isValidDate(day)) {
            cout << "Invalid date.\n";
            return;
        }

        vector<string> timeSlots = generateTimeSlots();
        map<string, string> schedule;

        for (const string& time : timeSlots) {
            schedule[time] = "Free";
        }

        if (events.find(day) != events.end()) {
            for (const auto& event : events[day]) {
                string currentTime = event.startTime;
                while (currentTime < event.endTime) {
                    schedule[currentTime] = event.title;
                    currentTime = addMinutes(currentTime, 30);
                }
            }
        }

        if (find(dayOffs.begin(), dayOffs.end(), day) != dayOffs.end()) {
            cout << "Day " << day << " is marked as day off.\n";
            return;
        }

        cout << "Events for day " << day << ":\n";
        for (const string& time : timeSlots) {
            string endTime = addMinutes(time, 30);
            cout << time << " - " << endTime << ": " << schedule[time] << "\n";
        }
    }

    void viewWeekSummary(int startDay) {
        if (!isValidDate(startDay) || startDay + 6 > DAYS_IN_JULY) {
            cout << "Invalid start day for week summary.\n";
            return;
        }

        for (int i = startDay; i < startDay + 7; ++i) {
            if (find(dayOffs.begin(), dayOffs.end(), i) != dayOffs.end()) {
                cout << "Day " << i << ": Day off\n";
            } else if (events.find(i) != events.end() && !events[i].empty()) {
                cout << "Day " << i << ":\n";
                for (const auto& event : events[i]) {
                    cout << "  " << event.startTime << " - " << event.endTime << ": " << event.title << "\n";
                }
            } else {
                cout << "Day " << i << ": No events\n";
            }
        }
    }

    void viewMonthSummary() {
        bool hasEventsOrDayOff = false;
        for (int i = 1; i <= DAYS_IN_JULY; ++i) {
            if (events.find(i) != events.end() && !events[i].empty()) {
                cout << "Day " << i << ":\n";
                for (const auto& event : events[i]) {
                    cout << "  " << event.startTime << " - " << event.endTime << ": " << event.title << "\n";
                }
                hasEventsOrDayOff = true;
            } else if (find(dayOffs.begin(), dayOffs.end(), i) != dayOffs.end()) {
                cout << "Day " << i << ": Day off\n";
                hasEventsOrDayOff = true;
            }
        }
        if (!hasEventsOrDayOff) {
            cout << "No scheduled events or day off days in July.\n";
        }
    }

    void deleteEvent(int day) {
        if (!isValidDate(day)) {
            cout << "Invalid date.\n";
            return;
        }

        if (events.find(day) == events.end() || events[day].empty()) {
            cout << "No events scheduled for this day to delete.\n";
            return;
        }

        cout << "Events for day " << day << ":\n";
        for (int i = 0; i < events[day].size(); ++i) {
            cout << i + 1 << ". " << events[day][i].startTime << " - " << events[day][i].endTime << ": " << events[day][i].title << "\n";
        }

        cout << "Enter event number to delete: ";
        int eventNumber;
        cin >> eventNumber;

        if (eventNumber < 1 || eventNumber > events[day].size()) {
            cout << "Invalid event number.\n";
            return;
        }

        events[day].erase(events[day].begin() + eventNumber - 1);
        cout << "Event deleted successfully.\n";
    }

    void editEvent(int day) {
        if (!isValidDate(day)) {
            cout << "Invalid date.\n";
            return;
        }

        if (events.find(day) == events.end() || events[day].empty()) {
            cout << "No events scheduled for this day to edit.\n";
            return;
        }

        cout << "Events for day " << day << ":\n";
        for (int i = 0; i < events[day].size(); ++i) {
            cout << i + 1 << ". " << events[day][i].startTime << " - " << events[day][i].endTime << ": " << events[day][i].title << "\n";
        }

        cout << "Enter event number to edit: ";
        int eventNumber;
        cin >> eventNumber;

        if (eventNumber < 1 || eventNumber > events[day].size()) {
            cout << "Invalid event number.\n";
            return;
        }

        Event& eventToEdit = events[day][eventNumber - 1];
        cout << "Editing event: " << eventToEdit.title << "\n";

        cout << "Enter new event title: ";
        cin.ignore();
        getline(cin, eventToEdit.title);

        cout << "Enter new event start time (HHMM): ";
        cin >> eventToEdit.startTime;
        if (!isValidTimeFormat(eventToEdit.startTime)) {
            cout << "Invalid time format.\n";
            return;
        }

        cout << "Enter new event end time (HHMM): ";
        cin >> eventToEdit.endTime;
        if (!isValidTimeFormat(eventToEdit.endTime) || eventToEdit.endTime <= eventToEdit.startTime) {
            cout << "Invalid end time format or end time is not after start time.\n";
            return;
        }

        if (isOvernightEvent(eventToEdit.startTime, eventToEdit.endTime)) {
            cout << "Overnight events are not allowed (between 22:00 and 06:00).\n";
            return;
        }

        cout << "Event edited successfully.\n";
    }

    void shiftEvent(int day) {
        if (!isValidDate(day)) {
            cout << "Invalid date.\n";
            return;
        }

        if (events.find(day) == events.end() || events[day].empty()) {
            cout << "No events scheduled for this day to shift.\n";
            return;
        }

        cout << "Events for day " << day << ":\n";
        for (int i = 0; i < events[day].size(); ++i) {
            cout << i + 1 << ". " << events[day][i].startTime << " - " << events[day][i].endTime << ": " << events[day][i].title << "\n";
        }

        cout << "Enter event number to shift: ";
        int eventNumber;
        cin >> eventNumber;

        if (eventNumber < 1 || eventNumber > events[day].size()) {
            cout << "Invalid event number.\n";
            return;
        }

        Event eventToShift = events[day][eventNumber - 1];

        int newDay;
        cout << "Enter new date (day): ";
        cin >> newDay;
        if (!isValidDate(newDay) || isDateInPast(newDay)) {
            cout << "Invalid date or date is in the past.\n";
            return;
        }

        if (isOvernightEvent(eventToShift.startTime, eventToShift.endTime)) {
            cout << "Overnight events are not allowed (between 22:00 and 06:00).\n";
            return;
        }

        for (const auto& event : events[newDay]) {
            if (isTimeOverlap(eventToShift, event)) {
                cout << "Event overlaps with an existing event on the new date.\n";
                return;
            }
        }

        events[newDay].push_back(eventToShift);
        events[day].erase(events[day].begin() + (eventNumber - 1));
        cout << "Event shifted successfully.\n";
    }
};

int main() {
    Calendar myCalendar;

    while (true) {
        cout << "\n============ Calendar Menu ============\n";
        cout << "1. Schedule Event\n";
        cout << "2. View Events by Date\n";
        cout << "3. View Week Summary\n";
        cout << "4. View Month Summary\n";
        cout << "5. Delete Event\n";
        cout << "6. Edit Event\n";
        cout << "7. Shift Event\n";
        cout << "8. Add Day Off\n";
        cout << "9. Exit\n";
        cout << "=======================================\n";
        cout << "Enter your choice: ";

        int choice;
        cin >> choice;

        switch (choice) {
            case 1:
                myCalendar.scheduleEvent();
                break;
            case 2: {
                int day;
                cout << "Enter day to view events: ";
                cin >> day;
                cout << "\n========= A Schedule for a Day ========\n";
                myCalendar.viewEventsByDate(day);
                break;
            }
            case 3: {
                int startDay;
                cout << "Enter start day of the week: ";
                cin >> startDay;
                cout << "\n======== A Schedule for a Week ========\n";
                myCalendar.viewWeekSummary(startDay);
                break;
            }
            case 4:
                cout << "\n=== Schedule for month of July 2024 ===\n";
                myCalendar.viewMonthSummary();
                break;
            case 5: {
                int dayToDelete;
                cout << "Enter day to delete event: ";
                cin >> dayToDelete;
                myCalendar.deleteEvent(dayToDelete);
                break;
            }
            case 6: {
                int dayToEdit;
                cout << "Enter day to edit event: ";
                cin >> dayToEdit;
                myCalendar.editEvent(dayToEdit);
                break;
            }
            case 7: {
                int dayToShift;
                cout << "Enter day to shift event: ";
                cin >> dayToShift;
                myCalendar.shiftEvent(dayToShift);
                break;
            }
            case 8: {
                int dayOff;
                cout << "Enter day to mark as day off: ";
                cin >> dayOff;
                myCalendar.addDayOff(dayOff);
                break;
            }
            case 9:
                cout << "Exiting...\n";
                return 0;
            default:
                cout << "Invalid choice. Please enter a number between 1 and 9.\n";
                break;
        }
    }

    return 0;
}
