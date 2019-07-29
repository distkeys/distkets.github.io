---
layout: post
title: "Designing BlackJack Game"
date: 2015-06-10 21:20
comments: true
categories:
 - Object Oriented Programming
 - C++

tags:
- '2015'
---


## Designing BlackJack Game

{% include toc %}

<br><br>
### Problem Statement

Design a blackjack game using object-oriented principles.

To come up with an object-oriented design for Black Jack game lets first go over the requirements of the system

- There is one dealer and multiple players.
- Each participant attempts to beat the dealer by getting a count as close to 21 as possible, without going over 21.
- The standard 52-card pack is used, but in most casinos, several decks of cards are shuffled together.
- An ace is worth 1 or 11.
- Before the deal begins, each player places a bet.
- When all the players have placed their bets, the dealer gives two card face up to each player in rotation clockwise, and then two card face up to himself.
- If a player's first two cards are an ace and a "ten-card" (a picture card or 10), giving him a count of 21 in two cards, this is a natural or "blackjack."
- After 1st round players can either fold the card, keep betting and get another card or bet and stand.
- After 1st round dealer will deal one card to all players who are in the game.
- Player whose total sum of cards is more than 21 automatically folds and lose the bet.



<br><br>
### Identify Classes and Relationships

![center-aligned-image](/images/blackjack.png)

<br><br>
### BlackJack - Header

{% highlight c++ linenos %}
#ifndef blackjack_hpp
#define blackjack_hpp

#include <stdio.h>
#include <string>
#include <vector>
#include <map>
#include <iostream>
#include <stdlib.h>

using namespace std;

enum SuitColor {DEFAULT=0, CLOVE, DIAMOND, HEART, SPADE};
enum PlayerType {DEALER=0, PLAYER};

/*************************
 * Class card
 *************************/
class card
{
private:
    SuitColor cardColor;
    int cardNumber;
    bool available;

public:
    card () {}
    card (SuitColor color, int number) {
        this->cardColor = color;
        this->cardNumber = number;
        this->available = true;
    }

    bool isAvailable() {
        return available;
    }

    int getCardNumber() {
        return cardNumber;
    }

    SuitColor getCardColor() {
        return cardColor;
    }
};

/*************************
 * Class cardDeck
 *************************/
class cardDeck
{
private:
    vector<card*> deck;

public:
    inline cardDeck();

    vector<card*> getDeck() {
        return deck;
    }
    card* draw();
    void shuffleDeck();
    void printDeck();
};

/*************************
 * Class Person
 *************************/
class person
{
protected:
    string name;
    int playerNumber;
    PlayerType type;
    int totalAmount;
    int amountLeft;
    int betAmount;
    bool bet;
    bool fold;
    bool stands;

public:
    person() {}
    person(string name, int totalAmount) {
        this->name = name;
        this->totalAmount = totalAmount;
        this->betAmount = 0;
    }
    virtual string getName() {
        return name;
    }

    virtual int getPlayerNumber() {
        return playerNumber;
    }

    virtual PlayerType getPlayerType() {
        return type;
    }

    virtual int getTotalAmount() {
        return totalAmount;
    }

    virtual  int getAmountLeft() {
        return amountLeft;
    }

    virtual int getBetAmount() {
        return betAmount;
    }

    virtual bool isFolding() {
        return false;
    }

    virtual void setFold(bool fold) {
        this->fold = false;
    }

    virtual bool didFold() {
        return fold;
    }

    virtual bool isStanding() {
        return false;
    }

    virtual void setStanding(bool stands) {
        this->stands = false;
    }

    virtual bool getStanding() {
        return this->stands;
    }

    virtual bool isBetting() {
        return true;
    }

    virtual bool getBetting() {
        return bet;
    }

    virtual bool canBet(int betAmount) {
        if (betAmount < amountLeft) {
            return true;
        }

        return false;
    }

    virtual void setBet(int betAmount) {
        amountLeft -= betAmount;
        this->betAmount += betAmount;
    }


    virtual void refillFunds(int amount) {
        totalAmount += amount;
        amountLeft += amount;
    }

    virtual int compensateLooseAmount() {
        return 0;
    }
    virtual void printObjectState() {}
};

/*************************
 * Class Dealer
 *************************/
class dealer:public person
{
private:
    cardDeck* deckObj;

public:
    dealer() {}
    dealer(string name, int totalAmount) {
        this->name = name;
        this->totalAmount = totalAmount;
        this->amountLeft = totalAmount;
        this->betAmount = 0;
        this->type = DEALER;

        // create deck of card. Deck belongs to dealer
        deckObj = new cardDeck();
    }

    card* deal();

    int compensateLooseAmount(int amount) {
        this->amountLeft -= amount;
        return amount;
    }

    void printDealerReport();
};

/*************************
 * Class player
 *************************/
class player:public person
{
public:
    player(string name, int totalAmount, int playerNumber) {
        this->name = name;
        this->totalAmount = totalAmount;
        this->amountLeft = totalAmount;
        this->betAmount = 0;
        this->type = PLAYER;

        fold = false;
        stands = false;
        bet = false;
        this->playerNumber = playerNumber;
    }
    bool flipCoin();
    bool isFolding();
    void setFold(bool fold);
    bool isStanding();
    void setStanding(bool stands);
    bool isBetting();
    bool canBet(int betAmount);
    int getAmountLeft();
    int compensateLooseAmount();
    void printPlayerReport();
};


/*************************
 * Class gameTable
 *************************/
class gameTable
{
private:
    map<person*, vector<card*>> hand;
    map<person*, int> scoreDetails;
    vector<person*> players;

public:
    gameTable() {}
    void addPlayerInHand(person *player);
    void setScore(person* player, int score);
    int getScore(person* player);
    void cardToPlayer(person* player, card* Card);
    vector<person*> getPlayers();
    void calculateScore();
    vector<person*> getPlayersHitBlackJack();
    void foldPlayersWhoLost(dealer *dealerObj);
    void removePlayerInHand(person *player);
    vector<person*> currentPlayersInGame();
    vector<card*> getPlayerCards(person* player);
    void printScoreReport();
};

void blackjackDriver();

#endif /* blackjack_hpp */

{% endhighlight %}



### BlackJack - Source

{% highlight c++ linenos %}
#include "blackjack.hpp"

/*************************
 * Class cardDeck methods
 *************************/
inline cardDeck::cardDeck()
{
    SuitColor color = DEFAULT;
    int colorCounter = 0;

    int cardNumber = 0;
    for (int i = 1; i <= 52; i++) {
        if (i % 13 == 1) {
            color = (SuitColor)(++colorCounter);
        }

        cardNumber = i % 13;
        cardNumber = (cardNumber == 0) ? 13 : cardNumber;

        card* cardObj = new card(color, cardNumber);
        deck.push_back(cardObj);
    }
    shuffleDeck();
}

card* cardDeck::draw()
{
    // Get card from deck
    if (!deck.empty()) {
        card* temp = deck.back();
        deck.pop_back();
        return temp;
    }

    return NULL;
}

void cardDeck::shuffleDeck()
{
    int indx = 0;
    card *temp;

    for (int i = 0; i <= 51; i++) {
        indx = rand() % 52;

        cout << "Swap " << i << ", " << indx << endl;
        temp = deck[i];
        deck[i] = deck[indx];
        deck[indx] = temp;
    }
}

void cardDeck::printDeck()
{
    for (int i = 0; i < deck.size(); i++) {
        switch (deck[i]->getCardColor()) {
            case CLOVE:
                cout << (i%13)+1 << " -> " << "CLOVE" << endl;
                break;

            case DIAMOND:
                cout << (i%13)+1 << " -> " << "DIAMOND" << endl;
                break;

            case HEART:
                cout << (i%13)+1 << " -> " << "HEART" << endl;
                break;

            case SPADE:
                cout << (i%13)+1 << " -> " << "SPADE" << endl;
                break;
            default:
                break;
        }
    }
}

/*************************
 * Class gameTable methods
 *************************/
void gameTable::addPlayerInHand(person *player) {
    hand.insert(std::pair<person*, vector<card*>>(player, NULL));
    scoreDetails.insert(std::pair<person*, int>(player, 0));

    this->players.push_back(player);
}

void gameTable::setScore(person* player, int score) {
    scoreDetails[player] = score;
}

int gameTable::getScore(person* player) {
    return scoreDetails[player];
}

void gameTable::cardToPlayer(person* player, card* Card) {
    map<person*, vector<card*>>::iterator handItr;

    handItr = hand.find(player);
    vector<card*> allCardsInHand = handItr->second;
    allCardsInHand.push_back(Card);
    handItr->second = allCardsInHand;
}

vector<person*> gameTable::getPlayers() {
    return players;
}

void gameTable::calculateScore() {
    // Go through each player and calulate score
    map<person*, vector<card*>>::iterator handItr;
    map<person*, int>::iterator scoreDetailsItr;

    for (handItr = hand.begin(); handItr != hand.end(); ++handItr) {
        vector<card*> allCardsSoFar = handItr->second;

        int totalscore = 0;
        for (int i = 0; i < allCardsSoFar.size(); i++) {
            switch(allCardsSoFar[i]->getCardNumber()) {
                    // Cards 10, J, Q, K
                case 10:
                case 11:
                case 12:
                case 13:
                    totalscore += 10;
                    break;
                case 1:
                    // Card 'A'
                    if (totalscore + 11 >= 21) {
                        totalscore += 1;
                    } else {
                        totalscore += 11;
                    }

                    break;
                default:
                    // All cards between 2-9
                    totalscore += allCardsSoFar[i]->getCardNumber();
                    break;
            }
        }
        scoreDetailsItr = scoreDetails.find(handItr->first);
        scoreDetailsItr->second = totalscore;
    }
}

vector<person*> gameTable::getPlayersHitBlackJack() {
    map<person*, int>::iterator scoreDetailsItr;
    vector<person*> temp;

    for (scoreDetailsItr = scoreDetails.begin();
         scoreDetailsItr != scoreDetails.end(); ++scoreDetailsItr) {
        if (scoreDetailsItr->second == 21 && !scoreDetailsItr->first->didFold() &&
            scoreDetailsItr->first->getPlayerType() == PLAYER) {
            temp.push_back(scoreDetailsItr->first);
        }
    }

    return temp;
}

void gameTable::removePlayerInHand(person *player) {
    hand.erase(player);
    scoreDetails.erase(player);

    vector<person*>::iterator playersItr;
    for (playersItr = players.begin(); playersItr != players.end(); ++playersItr) {
        if (*playersItr == player) {
            players.erase(playersItr);
        }
    }
}

vector<person*> gameTable::currentPlayersInGame()
{
    vector<person*> temp;
    for (int i = 0; i < players.size(); i++) {
        if (!players[i]->didFold()) {
            temp.push_back(players[i]);
        }
    }

    return temp;
}

vector<card*> gameTable::getPlayerCards(person* player)
{
    map<person*, vector<card*>>::iterator handItr;
    handItr = hand.find(player);

    return handItr->second;
}

void gameTable::foldPlayersWhoLost(dealer *dealerObj)
{
    map<person*, int>::iterator scoreDetailsItr;

    for (scoreDetailsItr = scoreDetails.begin();
         scoreDetailsItr != scoreDetails.end(); ++scoreDetailsItr) {
        if (scoreDetailsItr->first->getPlayerType() == PLAYER &&
            !scoreDetailsItr->first->didFold() &&
            scoreDetailsItr->second > 21) {

            // Get the bet amount of player
            dealerObj->refillFunds(scoreDetailsItr->first->getBetAmount());
            scoreDetailsItr->first->setFold(true);
        }
    }

    return;
}

void gameTable::printScoreReport()
{
    // Print score of each player
    cout << endl << "[Players Scores]" << endl;
    for (int i = 0; i < this->players.size(); i++) {
        cout << "   ==> [" << players[i]->getName() << "] " <<
        getScore(players[i]) << endl;
    }
}

/*************************
 * Class player methods
 *************************/
bool player::flipCoin() {
    return rand() % 2;
}

bool player::isFolding() {
    if (this->fold == true) {
        return true;
    }
    return flipCoin();
}

void player::setFold(bool fold) {
    this->fold = fold;
}

bool player::isStanding() {
    bool temp = flipCoin();
    setStanding(temp);
    return temp;
}

void player::setStanding(bool stands) {
    this->stands = stands;
}

bool player::isBetting() {
    return flipCoin();
}

bool player::canBet(int betAmount) {
    if (betAmount < this->amountLeft && this->fold == false) {
        return true;
    }

    return false;
}

int player::getAmountLeft() {
    return amountLeft;
}

int player::compensateLooseAmount() {
    int temp = betAmount;
    betAmount = 0;
    return temp;
}

void player::printPlayerReport()
{
    cout << "[Player]" << endl;
    cout << "   ==> Player Name: " << getName() << endl;
    cout << "   ==> Player Number: " << getPlayerNumber() << endl;
    cout << "   ==> Player Type: " << getPlayerType() << endl;
    cout << "   ==> Total Amount: " << getTotalAmount() << endl;
    cout << "   ==> Amount Left: " << getAmountLeft() << endl;
    cout << "   ==> Bet Amount: " << getBetAmount() << endl;
    cout << "   ==> Betting: " << getBetting() << endl;
    cout << "   ==> Fold: " << didFold() << endl;
    cout << "   ==> Stand: " << getStanding()  << endl << endl;
}

/*************************
 * Class dealer methods
 *************************/
card* dealer::deal()
{
    return deckObj->draw();
}

void dealer::printDealerReport()
{
    cout << "[Dealer]" << endl;
    cout << "   ==> Dealer Name: " << getName() << endl;
    cout << "   ==> Dealer Number: " << getPlayerNumber() << endl;
    cout << "   ==> Player Type: " << getPlayerType() << endl;
    cout << "   ==> Total Amount: " << getTotalAmount() << endl;
    cout << "   ==> Amount Left: " << getAmountLeft() << endl;
    cout << "   ==> Bet Amount: " << getBetAmount() << endl;
    cout << "   ==> Betting: " << getBetting() << endl;
    cout << "   ==> Fold: " << didFold() << endl;
    cout << "   ==> Stand: " << getStanding()  << endl << endl;
}

/*************************
 * blackjackDriver
 *************************/
void blackjackDriver()
{
    // Create dealer
    dealer dealerObj("Dealer", 10000);

    // Create players
    player player0("Player0", 200, 0);
    player player1("Player1", 200, 1);
    player player2("Player2", 200, 2);
    player player3("Player3", 200, 3);

    // Create Game Table
    gameTable gameObj;
    gameObj.addPlayerInHand(&player0);
    gameObj.addPlayerInHand(&player1);
    gameObj.addPlayerInHand(&player2);
    gameObj.addPlayerInHand(&player3);
    gameObj.addPlayerInHand(&dealerObj);

    // Print Player reports
    vector<person*> players = gameObj.getPlayers();
    dealerObj.printDealerReport();
    player0.printPlayerReport();
    player1.printPlayerReport();
    player2.printPlayerReport();
    player3.printPlayerReport();

    int round = 0;
    while(1) {
        round++;
        cout << endl << "========Round " << round << "========" << endl;

        // All players bet
        for (int i = 0; i < players.size(); i++) {
            if (players[i]->canBet(50)) {
                if (players[i]->getPlayerType() == PLAYER) {
                    players[i]->setBet(50);

                    cout << "[" << players[i]->getName() << "] betting: " <<
                                    players[i]->getBetAmount() << endl;
                }

                if (!players[i]->isStanding()) {
                    gameObj.cardToPlayer(players[i], dealerObj.deal());
                    if (round == 1) {
                        gameObj.cardToPlayer(players[i], dealerObj.deal());
                    }

                    cout << "[" << players[i]->getName() <<
                            "] is not standing" << endl;
                }

                vector<card*>cards =  gameObj.getPlayerCards(players[i]);
                for (vector<card*>::iterator cardsItr = cards.begin();
                     cardsItr != cards.end(); ++cardsItr) {
                    cout << "   ==> " << (*cardsItr)->getCardColor() <<
                            ", " << (*cardsItr)->getCardNumber() << endl;
                }
            }
        }

        // Find scores after each iteration
        gameObj.calculateScore();

        // Find players exceeded 21 and lost
        gameObj.foldPlayersWhoLost(&dealerObj);

        // Print score of each player
        gameObj.printScoreReport();

        // Find players hit blackJack and compensate them
        vector<person*> playersHitBlackJack = gameObj.getPlayersHitBlackJack();
        for (int i = 0; i < playersHitBlackJack.size(); i++) {
            int betAmount = playersHitBlackJack[i]->getBetAmount();

            // dealer compensate the amount to player
            playersHitBlackJack[i]->refillFunds(
                                dealerObj.compensateLooseAmount(2 * betAmount));
            playersHitBlackJack[i]->setFold(true);
        }

        // Find if dealer hit BJ or Over 21 or under 21?
        if (gameObj.getScore(&dealerObj) == 21) {
            // Game over and dealer get money from other players still in game
            for (int i = 0; i < playersHitBlackJack.size(); i++) {
                if (playersHitBlackJack[i]->getPlayerType() == PLAYER &&
                    !playersHitBlackJack[i]->didFold()) {
                    dealerObj.refillFunds(
                            playersHitBlackJack[i]->compensateLooseAmount());
                }
            }

            break;
        } else if (gameObj.getScore(&dealerObj) > 21) {
            // Dealer loose and other people get double the bet
            vector<person*> currentPlayers = gameObj.currentPlayersInGame();

            for (int i = 0; i < currentPlayers.size(); i++) {
                if (currentPlayers[i]->getPlayerType() == PLAYER &&
                    gameObj.getScore(currentPlayers[i]) < 21) {
                    int betAmount = currentPlayers[i]->getBetAmount();

                    currentPlayers[i]->refillFunds(
                                dealerObj.compensateLooseAmount(2 * betAmount));
                }
            }

            break;
        }
    }

    cout << endl << endl;
    cout << "****[Final Report]****" << endl;
    dealerObj.printDealerReport();
    player0.printPlayerReport();
    player1.printPlayerReport();
    player2.printPlayerReport();
    player3.printPlayerReport();

    cout << "========BlackJack Ended========";
    return;
}
{% endhighlight %}

<br><br><br><br>
