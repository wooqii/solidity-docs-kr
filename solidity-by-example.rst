###################
예제를 통한 솔리디티
###################

.. 색인:: voting, ballot

.. _voting:

******
투표 스마트계약
******

다음 계약은 확실히 복잡합니다. 그러나 다음 예시들은 솔리디티의
다양한 특징을 포함합니다. 이 예시는 투표계약을 구현합니다.
물론, 전자 투표가 당면한 핵심 문제는 올바른 사람에게 투표권을
부여하는 방법과 조작을 방지하는 방법입니다. 우리는 이 예시에서 
모든 문제를 해결하지는 않지만, 최소한 위임투표가 어떻게 
동시에 **자동적이며 완전 투명하게** 이루어지는지 보여줄 것입니다.


이 아이디어는 발의안 당 하나의 계약을 작성하여 각 옵션마다
짧은 이름을 붙입니다. 스마트계약에서 의장 역할을 수행하는 계약 작성자는 각 주소에 개별적인 투표권한을 부여할 것입니다.
그러면 의장으로 봉사하는 계약 작성자는 각 주소에 개별적으로 투표권을 부여합니다.

주소가 있는 사람들은 자신을 투표하거나 긴뢰할 수 있는 사람에게 투표를 위임할 수 있습니다.

투표시간이 끝나면, ``winningProposal()`` 이 가장 큰 숫자의 결과를 반환합니다.

::

    pragma solidity ^0.4.16;

    /// @title 대표단과 투표.
    contract Ballot {
        // 이것은 나중에 변수에 사용될 새로운
        // 복합 유형을 선언합니다.
        // 그것은 단일 유권자를 대표할 것입니다.
        struct Voter {
            uint weight; // weight 는 대표단에 의해 누적됩니다.
            bool voted;  // 만약 이 값이 true라면, 그 사람은 이미 투표한 것 입니다.
            address delegate; // 투표에 위임된 사람
            uint vote;   // 투표된 제안의 인덱스 데이터 값
        }

        // 이것은 단일 제안에 대한 유형입니다.
        struct Proposal {
            bytes32 name;   // 간단한 명칭 (최대 32바이트)
            uint voteCount; // 누적 투표 수
        }

        address public chairperson;

        // 이것은 각각의 가능한 주소에 대해
        // `Voter` 구조체를 저장하는 상태변수를 선언합니다.
        mapping(address => Voter) public voters;

        // 동적으로 크기가 지정된 `Proposal` 구조체의 배열입니다.
        Proposal[] public proposals;

        /// `proposalNames` 중 하나를 선택하기 위한 새로운 투표권을 생성십시오.
        function Ballot(bytes32[] proposalNames) public {
            chairperson = msg.sender;
            voters[chairperson].weight = 1;

            // 각각의 제공된 제안서 이름에 대해,
            // 새로운 제안서 개체를 만들어 배열 끝에 추가합니다.
            for (uint i = 0; i < proposalNames.length; i++) {
                // `Proposal({...})` creates a temporary
                // Proposal object and `proposals.push(...)`
                // appends it to the end of `proposals`.
                proposals.push(Proposal({
                    name: proposalNames[i],
                    voteCount: 0
                }));
            }
        }

        // `voter` 에게 이 투표권에 대한 권한을 부여하십시오.
        // 오직 `chairperson` 으로부터 호출받을 수 있습니다.
        function giveRightToVote(address voter) public {
            // `require`의 인수가 `false`로 평가되면,
            // 그것은 종료되고 모든 변경내용을 state와 
            // Ether Balance로 되돌립니다. 
            // 함수가 잘못 호출되면 이것을 사용하는 것이 좋습니다.
            // 그러나 조심하십시오,  
            // 이것은 현재 제공된 모든 가스를 소비할 것입니다.
            // (이것은 앞으로 바뀌게 될 예정입니다)
            require(
                (msg.sender == chairperson) &&
                !voters[voter].voted &&
                (voters[voter].weight == 0)
            );
            voters[voter].weight = 1;
        }

        /// `to` 로 유권자에게 투표를 위임하십시오.
        function delegate(address to) public {
            // 참조를 지정하십시오.
            Voter storage sender = voters[msg.sender];
            require(!sender.voted);

            // 자체 위임은 허용되지 않습니다.
            require(to != msg.sender);

            // `to`가 위임하는 동안 delegation을 전달하십시오.
            // 일반적으로 이런 루프는 매우 위험하기 때문에,
            // 너무 오래 실행되면 블록에서 사용가능한 가스보다
            // 더 많은 가스가 필요하게 될지도 모릅니다.
            // 이 경우 위임(delegation)은 실행되지 않지만,
            // 다른 상황에서는 이러한 루프로 인해
            // 스마트 계약서가 완전히 "고착"될 수 있습니다.
            while (voters[to].delegate != address(0)) {
                to = voters[to].delegate;

                // 우리는 delegation에 루프가 있음을 확인 했고 허용하지 않았습니다.
                require(to != msg.sender);
            }

            // `sender` 는 참조이므로, 
            // `voters[msg.sender].voted` 를 수정합니다.
            sender.voted = true;
            sender.delegate = to;
            Voter storage delegate_ = voters[to];
            if (delegate_.voted) {
                // 대표가 이미 투표한 경우,
                // 투표 수에 직접 추가 하십시오
                proposals[delegate_.vote].voteCount += sender.weight;
            } else {
                // 대표가 아직 투표하지 않았다면,
                // weight에 추가하십시오.
                delegate_.weight += sender.weight;
            }
        }

        /// (당신에게 위임된 투표권을 포함하여) 
        /// `proposals[proposal].name` 제안서에 투표 하십시오.
        function vote(uint proposal) public {
            Voter storage sender = voters[msg.sender];
            require(!sender.voted);
            sender.voted = true;
            sender.vote = proposal;

            // 만약 `proposal` 이 배열의 범위를 벗어나면
            // 자동으로 throw 하고 모든 변경사항을 되돌릴 것입니다.
            proposals[proposal].voteCount += sender.weight;
        }

        /// @dev 모든 이전 득표를 고려하여 승리한 제안서를 계산합니다.
        function winningProposal() public view
                returns (uint winningProposal_)
        {
            uint winningVoteCount = 0;
            for (uint p = 0; p < proposals.length; p++) {
                if (proposals[p].voteCount > winningVoteCount) {
                    winningVoteCount = proposals[p].voteCount;
                    winningProposal_ = p;
                }
            }
        }

        // winningProposal() 함수를 호출하여 
        // 제안 배열에 포함된 승자의 index를 가져온 다음
        // 승자의 이름을 반환합니다.
        function winnerName() public view
                returns (bytes32 winnerName_)
        {
            winnerName_ = proposals[winningProposal()].name;
        }
    }


개선가능 한 사안들
=====================

현재 모든 참가자에게 투표권을 부여하기 위해 많은 거래가 필요 합니다.
더 나은 방법을 생각해 볼 수 있습니까?

.. 색인:: auction;blind, auction;open, blind auction, open auction

*************
비공개 경매
*************

이번 섹션에서 우리는 완전한 비공개 경매 스마트 컨트랙트를
이더리움 상에서 쉽게 구현하는 방법을 보여줄 것입니다.
우선 우리는 모든 사람이 서로의 입찰가를 알 수 있는
공개 경매부터 시작하여 경매 기한이 종료될 때까지 
서로의 입찰가를 알 수 없는 비공개 경매로 예제를 확장할 것입니다.

.. _simple_auction:

단순 공개 경매
===================

단순 경매 컨트랙트의 일반적인 개념은 모든 이가 
경매 기간동안 자신의 입찰가를 입력하는 것 입니다.
이 입찰은 먼저 현금이나 이더를 전송하는 것으로 입찰자가 
자신의 경매를 되돌릴 수 없게 합니다. 만일 가장 높은 입찰가가 들어온다면, 이전의 최고 입찰가는 입찰자에게 환불 됩니다.  
입찰 기간이 종료된 후에, 스마트 컨트랙트는 수동으로 호출되어 수혜자에게 돈을 돌려주게 됩니다. - 스마트 컨트랙트는 스스로 활성화할 수 없습니다.

::

    pragma solidity ^0.4.21;

    contract SimpleAuction {
        // 경매의 parameter 입니다. 
        // 시간은 절대 유닉스 타임스탬프 ( 1970-01-01 이후의 초 단위) 입니다.
        // address 유형의 beneficiary 변수를 public으로 선언했습니다.
        address public beneficiary;
        // uint 타입의 auctionEnd 변수를 public으로 선언했습니다.
        uint public auctionEnd;

        // 현재 경매의 상태를 나타냅니다.
        // address 타입의 highestBidder를 public으로 선언하여 가장 높이 입찰한 사람의 주소를 나타냅니다.
        address public highestBidder;
        // uint 타입의 highestBid를 public으로 선언하여 최고입찰가를 나타냅니다.
        uint public highestBid;

        // 최고입찰가 이전의 입찰가를 인출할 수 있도록 허용합니다.
        // addrss를 키로 하고 uint를 반환하는 pendingReturns를 맵핑합니다.
        mapping(address => uint) pendingReturns;

        // 마지막에 true를 설정하여 변경을 허용하지 않게 합니다.
        bool ended;

        // 특정 변경점이 생길 시 이벤트를 발생시킵니다. 
        // HighestBidIncreased 라는 이벤트는 address타입의 bidder와 정수타입의 amount를 가진다. 
        event HighestBidIncreased(address bidder, uint amount);
        // AuctionEnded 라는 이벤트는 address타입의 winner와 정수타입의 amount를 가진다.
        event AuctionEnded(address winner, uint amount);

        // 다음의 것은  natspec comment,라고  불린다.
        // recognizable by the three slashes.
        // It will be shown when the user is asked to
        // confirm a transaction.

        /// Create a simple auction with `_biddingTime`
        /// seconds bidding time on behalf of the
        /// beneficiary address `_beneficiary`.
        function SimpleAuction(
            uint _biddingTime,
            address _beneficiary
        ) public {
            beneficiary = _beneficiary;
            auctionEnd = now + _biddingTime;
        }

        /// Bid on the auction with the value sent
        /// together with this transaction.
        /// The value will only be refunded if the
        /// auction is not won.
        function bid() public payable {
            // No arguments are necessary, all
            // information is already part of
            // the transaction. The keyword payable
            // is required for the function to
            // be able to receive Ether.

            // Revert the call if the bidding
            // period is over.
            require(now <= auctionEnd);

            // If the bid is not higher, send the
            // money back.
            require(msg.value > highestBid);

            if (highestBid != 0) {
                // Sending back the money by simply using
                // highestBidder.send(highestBid) is a security risk
                // because it could execute an untrusted contract.
                // It is always safer to let the recipients
                // withdraw their money themselves.
                pendingReturns[highestBidder] += highestBid;
            }
            highestBidder = msg.sender;
            highestBid = msg.value;
            emit HighestBidIncreased(msg.sender, msg.value);
        }

        /// Withdraw a bid that was overbid.
        function withdraw() public returns (bool) {
            uint amount = pendingReturns[msg.sender];
            if (amount > 0) {
                // It is important to set this to zero because the recipient
                // can call this function again as part of the receiving call
                // before `send` returns.
                pendingReturns[msg.sender] = 0;

                if (!msg.sender.send(amount)) {
                    // No need to call throw here, just reset the amount owing
                    pendingReturns[msg.sender] = amount;
                    return false;
                }
            }
            return true;
        }

        /// End the auction and send the highest bid
        /// to the beneficiary.
        function auctionEnd() public {
            // It is a good guideline to structure functions that interact
            // with other contracts (i.e. they call functions or send Ether)
            // into three phases:
            // 1. checking conditions
            // 2. performing actions (potentially changing conditions)
            // 3. interacting with other contracts
            // If these phases are mixed up, the other contract could call
            // back into the current contract and modify the state or cause
            // effects (ether payout) to be performed multiple times.
            // If functions called internally include interaction with external
            // contracts, they also have to be considered interaction with
            // external contracts.

            // 1. Conditions
            require(now >= auctionEnd); // auction did not yet end
            require(!ended); // this function has already been called

            // 2. Effects
            ended = true;
            emit AuctionEnded(highestBidder, highestBid);

            // 3. Interaction
            beneficiary.transfer(highestBid);
        }
    }

Blind Auction
=============

The previous open auction is extended to a blind auction
in the following. The advantage of a blind auction is
that there is no time pressure towards the end of
the bidding period. Creating a blind auction on a
transparent computing platform might sound like a
contradiction, but cryptography comes to the rescue.

During the **bidding period**, a bidder does not
actually send her bid, but only a hashed version of it.
Since it is currently considered practically impossible
to find two (sufficiently long) values whose hash
values are equal, the bidder commits to the bid by that.
After the end of the bidding period, the bidders have
to reveal their bids: They send their values
unencrypted and the contract checks that the hash value
is the same as the one provided during the bidding period.

Another challenge is how to make the auction
**binding and blind** at the same time: The only way to
prevent the bidder from just not sending the money
after he won the auction is to make her send it
together with the bid. Since value transfers cannot
be blinded in Ethereum, anyone can see the value.

The following contract solves this problem by
accepting any value that is larger than the highest
bid. Since this can of course only be checked during
the reveal phase, some bids might be **invalid**, and
this is on purpose (it even provides an explicit
flag to place invalid bids with high value transfers):
Bidders can confuse competition by placing several
high or low invalid bids.


::

    pragma solidity ^0.4.21;

    contract BlindAuction {
        struct Bid {
            bytes32 blindedBid;
            uint deposit;
        }

        address public beneficiary;
        uint public biddingEnd;
        uint public revealEnd;
        bool public ended;

        mapping(address => Bid[]) public bids;

        address public highestBidder;
        uint public highestBid;

        // Allowed withdrawals of previous bids
        mapping(address => uint) pendingReturns;

        event AuctionEnded(address winner, uint highestBid);

        /// Modifiers are a convenient way to validate inputs to
        /// functions. `onlyBefore` is applied to `bid` below:
        /// The new function body is the modifier's body where
        /// `_` is replaced by the old function body.
        modifier onlyBefore(uint _time) { require(now < _time); _; }
        modifier onlyAfter(uint _time) { require(now > _time); _; }

        function BlindAuction(
            uint _biddingTime,
            uint _revealTime,
            address _beneficiary
        ) public {
            beneficiary = _beneficiary;
            biddingEnd = now + _biddingTime;
            revealEnd = biddingEnd + _revealTime;
        }

        /// Place a blinded bid with `_blindedBid` = keccak256(value,
        /// fake, secret).
        /// The sent ether is only refunded if the bid is correctly
        /// revealed in the revealing phase. The bid is valid if the
        /// ether sent together with the bid is at least "value" and
        /// "fake" is not true. Setting "fake" to true and sending
        /// not the exact amount are ways to hide the real bid but
        /// still make the required deposit. The same address can
        /// place multiple bids.
        function bid(bytes32 _blindedBid)
            public
            payable
            onlyBefore(biddingEnd)
        {
            bids[msg.sender].push(Bid({
                blindedBid: _blindedBid,
                deposit: msg.value
            }));
        }

        /// Reveal your blinded bids. You will get a refund for all
        /// correctly blinded invalid bids and for all bids except for
        /// the totally highest.
        function reveal(
            uint[] _values,
            bool[] _fake,
            bytes32[] _secret
        )
            public
            onlyAfter(biddingEnd)
            onlyBefore(revealEnd)
        {
            uint length = bids[msg.sender].length;
            require(_values.length == length);
            require(_fake.length == length);
            require(_secret.length == length);

            uint refund;
            for (uint i = 0; i < length; i++) {
                var bid = bids[msg.sender][i];
                var (value, fake, secret) =
                        (_values[i], _fake[i], _secret[i]);
                if (bid.blindedBid != keccak256(value, fake, secret)) {
                    // Bid was not actually revealed.
                    // Do not refund deposit.
                    continue;
                }
                refund += bid.deposit;
                if (!fake && bid.deposit >= value) {
                    if (placeBid(msg.sender, value))
                        refund -= value;
                }
                // Make it impossible for the sender to re-claim
                // the same deposit.
                bid.blindedBid = bytes32(0);
            }
            msg.sender.transfer(refund);
        }

        // This is an "internal" function which means that it
        // can only be called from the contract itself (or from
        // derived contracts).
        function placeBid(address bidder, uint value) internal
                returns (bool success)
        {
            if (value <= highestBid) {
                return false;
            }
            if (highestBidder != 0) {
                // Refund the previously highest bidder.
                pendingReturns[highestBidder] += highestBid;
            }
            highestBid = value;
            highestBidder = bidder;
            return true;
        }

        /// Withdraw a bid that was overbid.
        function withdraw() public {
            uint amount = pendingReturns[msg.sender];
            if (amount > 0) {
                // It is important to set this to zero because the recipient
                // can call this function again as part of the receiving call
                // before `transfer` returns (see the remark above about
                // conditions -> effects -> interaction).
                pendingReturns[msg.sender] = 0;

                msg.sender.transfer(amount);
            }
        }

        /// End the auction and send the highest bid
        /// to the beneficiary.
        function auctionEnd()
            public
            onlyAfter(revealEnd)
        {
            require(!ended);
            emit AuctionEnded(highestBidder, highestBid);
            ended = true;
            beneficiary.transfer(highestBid);
        }
    }


.. index:: purchase, remote purchase, escrow

********************
Safe Remote Purchase
********************

::

    pragma solidity ^0.4.21;

    contract Purchase {
        uint public value;
        address public seller;
        address public buyer;
        enum State { Created, Locked, Inactive }
        State public state;

        // Ensure that `msg.value` is an even number.
        // Division will truncate if it is an odd number.
        // Check via multiplication that it wasn't an odd number.
        function Purchase() public payable {
            seller = msg.sender;
            value = msg.value / 2;
            require((2 * value) == msg.value);
        }

        modifier condition(bool _condition) {
            require(_condition);
            _;
        }

        modifier onlyBuyer() {
            require(msg.sender == buyer);
            _;
        }

        modifier onlySeller() {
            require(msg.sender == seller);
            _;
        }

        modifier inState(State _state) {
            require(state == _state);
            _;
        }

        event Aborted();
        event PurchaseConfirmed();
        event ItemReceived();

        /// Abort the purchase and reclaim the ether.
        /// Can only be called by the seller before
        /// the contract is locked.
        function abort()
            public
            onlySeller
            inState(State.Created)
        {
            emit Aborted();
            state = State.Inactive;
            seller.transfer(this.balance);
        }

        /// Confirm the purchase as buyer.
        /// Transaction has to include `2 * value` ether.
        /// The ether will be locked until confirmReceived
        /// is called.
        function confirmPurchase()
            public
            inState(State.Created)
            condition(msg.value == (2 * value))
            payable
        {
            emit PurchaseConfirmed();
            buyer = msg.sender;
            state = State.Locked;
        }

        /// Confirm that you (the buyer) received the item.
        /// This will release the locked ether.
        function confirmReceived()
            public
            onlyBuyer
            inState(State.Locked)
        {
            emit ItemReceived();
            // It is important to change the state first because
            // otherwise, the contracts called using `send` below
            // can call in again here.
            state = State.Inactive;

            // NOTE: This actually allows both the buyer and the seller to
            // block the refund - the withdraw pattern should be used.

            buyer.transfer(value);
            seller.transfer(this.balance);
        }
    }

********************
Micropayment Channel
********************

To be written.
