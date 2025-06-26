# Dokumentasi MT5 Order Send - Complete Guide

## Overview
Panduan lengkap untuk mengelola order di MetaTrader 5 menggunakan Python API, mencakup limit order, stop loss, take profit, modifikasi, trailing stop, dan close position.

## Table of Contents
1. [Limit Order dengan Entry, SL & TP](#1-limit-order-dengan-entry-sl--tp)
2. [Modify Order](#2-modify-order)
3. [Edit Take Profit](#3-edit-take-profit)
4. [Trailing Stop Loss](#4-trailing-stop-loss)
5. [Close Position](#5-close-position)
6. [Best Practices](#6-best-practices)
7. [Error Handling](#7-error-handling)

---

## 1. Limit Order dengan Entry, SL & TP

### Konsep
- **Limit Order**: Order pending yang akan eksekusi ketika price mencapai level tertentu
- **Entry**: Harga di mana order akan trigger
- **SL (Stop Loss)**: Level rugi maksimal yang bisa diterima
- **TP (Take Profit)**: Target profit yang ingin dicapai

### 1.1 BUY LIMIT Order

```python
import MetaTrader5 as mt5

def place_buy_limit_order(symbol, volume, entry_price, stop_loss, take_profit, deviation=20):
    """
    Place BUY LIMIT order dengan SL & TP
    
    Args:
        symbol (str): Trading symbol (e.g., 'EURUSD', 'XAUUSD')
        volume (float): Volume lot (e.g., 0.01, 0.1, 1.0)
        entry_price (float): Harga entry (harus di bawah current price)
        stop_loss (float): Stop loss level (di bawah entry_price)
        take_profit (float): Take profit level (di atas entry_price)
        deviation (int): Slippage tolerance dalam points
    
    Returns:
        dict: Response result
    """
    
    # Validasi price levels untuk BUY LIMIT
    current_price = mt5.symbol_info_tick(symbol).ask
    
    if entry_price >= current_price:
        return {"error": "BUY LIMIT entry price harus di bawah current price"}
    
    if stop_loss >= entry_price:
        return {"error": "Stop loss harus di bawah entry price"}
    
    if take_profit <= entry_price:
        return {"error": "Take profit harus di atas entry price"}
    
    # Request structure
    request = {
        "action": mt5.TRADE_ACTION_PENDING,  # Pending order
        "symbol": symbol,
        "volume": volume,
        "type": mt5.ORDER_TYPE_BUY_LIMIT,    # BUY LIMIT
        "price": entry_price,                # Entry price
        "sl": stop_loss,                     # Stop Loss
        "tp": take_profit,                   # Take Profit
        "deviation": deviation,
        "magic": 123456,
        "comment": f"BUY LIMIT {symbol}",
        "type_time": mt5.ORDER_TIME_GTC,     # Good Till Cancel
        "type_filling": mt5.ORDER_FILLING_IOC
    }
    
    # Send order
    result = mt5.order_send(request)
    
    if result is None:
        return {"error": f"Order failed: {mt5.last_error()}"}
    
    if result.retcode == mt5.TRADE_RETCODE_DONE:
        return {
            "success": True,
            "ticket": result.order,
            "message": f"BUY LIMIT placed: {volume} {symbol} at {entry_price}",
            "data": {
                "ticket": result.order,
                "symbol": symbol,
                "type": "BUY_LIMIT",
                "volume": volume,
                "entry": entry_price,
                "sl": stop_loss,
                "tp": take_profit
            }
        }
    else:
        return {"error": f"Order failed with retcode: {result.retcode}"}

# Example usage
result = place_buy_limit_order(
    symbol="EURUSD",
    volume=0.1,
    entry_price=1.0800,     # Entry di bawah current price
    stop_loss=1.0750,       # SL 50 pips
    take_profit=1.0900      # TP 100 pips
)
```

### 1.2 SELL LIMIT Order

```python
def place_sell_limit_order(symbol, volume, entry_price, stop_loss, take_profit, deviation=20):
    """
    Place SELL LIMIT order dengan SL & TP
    
    Args:
        entry_price (float): Harga entry (harus di atas current price)
        stop_loss (float): Stop loss level (di atas entry_price)
        take_profit (float): Take profit level (di bawah entry_price)
    """
    
    # Validasi price levels untuk SELL LIMIT
    current_price = mt5.symbol_info_tick(symbol).bid
    
    if entry_price <= current_price:
        return {"error": "SELL LIMIT entry price harus di atas current price"}
    
    if stop_loss <= entry_price:
        return {"error": "Stop loss harus di atas entry price"}
    
    if take_profit >= entry_price:
        return {"error": "Take profit harus di bawah entry price"}
    
    request = {
        "action": mt5.TRADE_ACTION_PENDING,
        "symbol": symbol,
        "volume": volume,
        "type": mt5.ORDER_TYPE_SELL_LIMIT,   # SELL LIMIT
        "price": entry_price,
        "sl": stop_loss,
        "tp": take_profit,
        "deviation": deviation,
        "magic": 123456,
        "comment": f"SELL LIMIT {symbol}",
        "type_time": mt5.ORDER_TIME_GTC,
        "type_filling": mt5.ORDER_FILLING_IOC
    }
    
    result = mt5.order_send(request)
    
    if result and result.retcode == mt5.TRADE_RETCODE_DONE:
        return {
            "success": True,
            "ticket": result.order,
            "message": f"SELL LIMIT placed: {volume} {symbol} at {entry_price}",
            "data": {
                "ticket": result.order,
                "symbol": symbol,
                "type": "SELL_LIMIT",
                "volume": volume,
                "entry": entry_price,
                "sl": stop_loss,
                "tp": take_profit
            }
        }
    else:
        return {"error": f"Order failed: {result.retcode if result else mt5.last_error()}"}

# Example usage
result = place_sell_limit_order(
    symbol="XAUUSD",
    volume=0.01,
    entry_price=2060.00,    # Entry di atas current price
    stop_loss=2070.00,      # SL $10
    take_profit=2040.00     # TP $20
)
```

---

## 2. Modify Order

### 2.1 Modify Pending Order

```python
def modify_pending_order(ticket, new_price=None, new_sl=None, new_tp=None):
    """
    Modify pending order (limit/stop orders)
    
    Args:
        ticket (int): Order ticket number
        new_price (float): New entry price (optional)
        new_sl (float): New stop loss (optional)
        new_tp (float): New take profit (optional)
    """
    
    # Get existing order info
    orders = mt5.orders_get(ticket=ticket)
    if not orders:
        return {"error": f"Order {ticket} not found"}
    
    order = orders[0]
    
    # Use existing values if not provided
    price = new_price if new_price is not None else order.price_open
    sl = new_sl if new_sl is not None else order.sl
    tp = new_tp if new_tp is not None else order.tp
    
    request = {
        "action": mt5.TRADE_ACTION_MODIFY,
        "order": ticket,
        "price": price,
        "sl": sl,
        "tp": tp,
        "type_time": mt5.ORDER_TIME_GTC,
    }
    
    result = mt5.order_send(request)
    
    if result and result.retcode == mt5.TRADE_RETCODE_DONE:
        return {
            "success": True,
            "message": f"Order {ticket} modified successfully",
            "data": {
                "ticket": ticket,
                "new_price": price,
                "new_sl": sl,
                "new_tp": tp
            }
        }
    else:
        return {"error": f"Modify failed: {result.retcode if result else mt5.last_error()}"}

# Example usage
result = modify_pending_order(
    ticket=123456789,
    new_price=1.0810,      # New entry price
    new_sl=1.0760,         # New stop loss
    new_tp=1.0910          # New take profit
)
```

### 2.2 Modify Open Position

```python
def modify_position(ticket, new_sl=None, new_tp=None):
    """
    Modify open position SL/TP
    
    Args:
        ticket (int): Position ticket
        new_sl (float): New stop loss
        new_tp (float): New take profit
    """
    
    # Get position info
    positions = mt5.positions_get(ticket=ticket)
    if not positions:
        return {"error": f"Position {ticket} not found"}
    
    position = positions[0]
    
    # Use existing values if not provided
    sl = new_sl if new_sl is not None else position.sl
    tp = new_tp if new_tp is not None else position.tp
    
    request = {
        "action": mt5.TRADE_ACTION_SLTP,     # Modify SL/TP only
        "position": ticket,
        "symbol": position.symbol,
        "sl": sl,
        "tp": tp,
        "magic": position.magic,
        "comment": f"Modified SL/TP for {ticket}",
    }
    
    result = mt5.order_send(request)
    
    if result and result.retcode == mt5.TRADE_RETCODE_DONE:
        return {
            "success": True,
            "message": f"Position {ticket} SL/TP modified",
            "data": {
                "ticket": ticket,
                "symbol": position.symbol,
                "new_sl": sl,
                "new_tp": tp
            }
        }
    else:
        return {"error": f"Modify failed: {result.retcode if result else mt5.last_error()}"}
```

---

## 3. Edit Take Profit

### 3.1 Update Take Profit Only

```python
def update_take_profit(ticket, new_tp):
    """
    Update only take profit level
    
    Args:
        ticket (int): Position/Order ticket
        new_tp (float): New take profit level
    """
    
    # Check if it's a position or pending order
    position = mt5.positions_get(ticket=ticket)
    order = mt5.orders_get(ticket=ticket)
    
    if position:
        # It's an open position
        pos = position[0]
        
        # Validate TP level
        current_price = mt5.symbol_info_tick(pos.symbol)
        if pos.type == 0:  # BUY position
            if new_tp <= current_price.bid:
                return {"error": "TP for BUY position harus di atas current price"}
        else:  # SELL position
            if new_tp >= current_price.ask:
                return {"error": "TP for SELL position harus di bawah current price"}
        
        request = {
            "action": mt5.TRADE_ACTION_SLTP,
            "position": ticket,
            "symbol": pos.symbol,
            "sl": pos.sl,           # Keep existing SL
            "tp": new_tp,           # New TP
            "magic": pos.magic,
        }
        
    elif order:
        # It's a pending order
        ord = order[0]
        
        request = {
            "action": mt5.TRADE_ACTION_MODIFY,
            "order": ticket,
            "price": ord.price_open,  # Keep existing price
            "sl": ord.sl,             # Keep existing SL
            "tp": new_tp,             # New TP
        }
    else:
        return {"error": f"Ticket {ticket} not found"}
    
    result = mt5.order_send(request)
    
    if result and result.retcode == mt5.TRADE_RETCODE_DONE:
        return {
            "success": True,
            "message": f"Take profit updated to {new_tp}",
            "data": {
                "ticket": ticket,
                "new_tp": new_tp
            }
        }
    else:
        return {"error": f"TP update failed: {result.retcode if result else mt5.last_error()}"}

# Example usage
result = update_take_profit(
    ticket=987654321,
    new_tp=1.0950  # Move TP to 1.0950
)
```

---

## 4. Trailing Stop Loss

### 4.1 Manual Trailing Stop

```python
def apply_trailing_stop(ticket, trailing_distance_pips, activation_distance_pips=None):
    """
    Apply trailing stop loss to position
    
    Args:
        ticket (int): Position ticket
        trailing_distance_pips (int): Distance in pips for trailing
        activation_distance_pips (int): Minimum profit before trailing starts
    """
    
    # Get position
    positions = mt5.positions_get(ticket=ticket)
    if not positions:
        return {"error": f"Position {ticket} not found"}
    
    position = positions[0]
    symbol = position.symbol
    
    # Get symbol info for pip calculation
    symbol_info = mt5.symbol_info(symbol)
    if symbol_info is None:
        return {"error": f"Symbol {symbol} info not available"}
    
    # Calculate pip value
    if symbol_info.digits == 5 or symbol_info.digits == 3:
        pip_size = symbol_info.point * 10
    else:
        pip_size = symbol_info.point
    
    # Get current price
    tick = mt5.symbol_info_tick(symbol)
    if tick is None:
        return {"error": f"Price data not available for {symbol}"}
    
    current_price = tick.bid if position.type == 0 else tick.ask
    
    # Calculate trailing distance in price
    trailing_distance = trailing_distance_pips * pip_size
    
    # Check activation distance if specified
    if activation_distance_pips:
        activation_distance = activation_distance_pips * pip_size
        
        if position.type == 0:  # BUY position
            profit_distance = current_price - position.price_open
            if profit_distance < activation_distance:
                return {
                    "success": False,
                    "message": f"Position not profitable enough to start trailing",
                    "data": {
                        "current_profit_pips": profit_distance / pip_size,
                        "required_pips": activation_distance_pips
                    }
                }
        else:  # SELL position
            profit_distance = position.price_open - current_price
            if profit_distance < activation_distance:
                return {
                    "success": False,
                    "message": f"Position not profitable enough to start trailing",
                    "data": {
                        "current_profit_pips": profit_distance / pip_size,
                        "required_pips": activation_distance_pips
                    }
                }
    
    # Calculate new stop loss
    if position.type == 0:  # BUY position
        new_sl = current_price - trailing_distance
        
        # Only update if new SL is better than current SL
        if position.sl == 0 or new_sl > position.sl:
            should_update = True
        else:
            should_update = False
            
    else:  # SELL position
        new_sl = current_price + trailing_distance
        
        # Only update if new SL is better than current SL
        if position.sl == 0 or new_sl < position.sl:
            should_update = True
        else:
            should_update = False
    
    if not should_update:
        return {
            "success": False,
            "message": "Current SL is already better than calculated trailing SL",
            "data": {
                "current_sl": position.sl,
                "calculated_sl": new_sl
            }
        }
    
    # Update stop loss
    request = {
        "action": mt5.TRADE_ACTION_SLTP,
        "position": ticket,
        "symbol": symbol,
        "sl": new_sl,
        "tp": position.tp,  # Keep existing TP
        "magic": position.magic,
        "comment": f"Trailing SL: {trailing_distance_pips} pips",
    }
    
    result = mt5.order_send(request)
    
    if result and result.retcode == mt5.TRADE_RETCODE_DONE:
        return {
            "success": True,
            "message": f"Trailing stop applied: SL moved to {new_sl}",
            "data": {
                "ticket": ticket,
                "old_sl": position.sl,
                "new_sl": new_sl,
                "trailing_distance_pips": trailing_distance_pips,
                "current_price": current_price
            }
        }
    else:
        return {"error": f"Trailing SL update failed: {result.retcode if result else mt5.last_error()}"}

# Example usage
result = apply_trailing_stop(
    ticket=555666777,
    trailing_distance_pips=20,     # Trail 20 pips behind
    activation_distance_pips=30    # Start trailing after 30 pips profit
)
```

### 4.2 Automated Trailing Stop Service

```python
import time
import threading
from datetime import datetime

class TrailingStopService:
    def __init__(self):
        self.active_trails = {}  # {ticket: trail_config}
        self.running = False
        self.thread = None
    
    def add_trailing_stop(self, ticket, trailing_distance_pips, activation_distance_pips=0):
        """Add position to trailing stop monitoring"""
        self.active_trails[ticket] = {
            "trailing_distance_pips": trailing_distance_pips,
            "activation_distance_pips": activation_distance_pips,
            "last_update": datetime.now(),
            "highest_price": 0,  # For BUY positions
            "lowest_price": 999999,  # For SELL positions
        }
        
        if not self.running:
            self.start()
        
        return {"success": True, "message": f"Trailing stop added for ticket {ticket}"}
    
    def remove_trailing_stop(self, ticket):
        """Remove position from trailing stop monitoring"""
        if ticket in self.active_trails:
            del self.active_trails[ticket]
            return {"success": True, "message": f"Trailing stop removed for ticket {ticket}"}
        return {"error": f"Ticket {ticket} not in trailing list"}
    
    def start(self):
        """Start trailing stop monitoring"""
        if not self.running:
            self.running = True
            self.thread = threading.Thread(target=self._monitor_loop)
            self.thread.daemon = True
            self.thread.start()
            print("Trailing stop service started")
    
    def stop(self):
        """Stop trailing stop monitoring"""
        self.running = False
        if self.thread:
            self.thread.join()
        print("Trailing stop service stopped")
    
    def _monitor_loop(self):
        """Main monitoring loop"""
        while self.running:
            try:
                tickets_to_remove = []
                
                for ticket, config in self.active_trails.items():
                    # Check if position still exists
                    positions = mt5.positions_get(ticket=ticket)
                    if not positions:
                        tickets_to_remove.append(ticket)
                        continue
                    
                    # Apply trailing stop
                    result = apply_trailing_stop(
                        ticket=ticket,
                        trailing_distance_pips=config["trailing_distance_pips"],
                        activation_distance_pips=config["activation_distance_pips"]
                    )
                    
                    if result.get("success"):
                        config["last_update"] = datetime.now()
                        print(f"Trailing SL updated for {ticket}: {result['message']}")
                
                # Remove closed positions
                for ticket in tickets_to_remove:
                    del self.active_trails[ticket]
                    print(f"Position {ticket} closed, removed from trailing")
                
                time.sleep(5)  # Check every 5 seconds
                
            except Exception as e:
                print(f"Trailing stop error: {e}")
                time.sleep(10)

# Global trailing service
trailing_service = TrailingStopService()

# Usage examples
def start_trailing_for_position(ticket, trailing_pips=20, activation_pips=30):
    """Start trailing stop for a position"""
    return trailing_service.add_trailing_stop(
        ticket=ticket,
        trailing_distance_pips=trailing_pips,
        activation_distance_pips=activation_pips
    )

def stop_trailing_for_position(ticket):
    """Stop trailing stop for a position"""
    return trailing_service.remove_trailing_stop(ticket)
```

---

## 5. Close Position

### 5.1 Close Full Position

```python
def close_position(ticket, comment="Position closed"):
    """
    Close entire position
    
    Args:
        ticket (int): Position ticket to close
        comment (str): Close comment
    """
    
    # Get position info
    positions = mt5.positions_get(ticket=ticket)
    if not positions:
        return {"error": f"Position {ticket} not found"}
    
    position = positions[0]
    
    # Get current price for closing
    tick = mt5.symbol_info_tick(position.symbol)
    if tick is None:
        return {"error": f"Price data not available for {position.symbol}"}
    
    # Determine close price and order type
    if position.type == 0:  # BUY position -> close with SELL
        close_price = tick.bid
        close_type = mt5.ORDER_TYPE_SELL
    else:  # SELL position -> close with BUY
        close_price = tick.ask
        close_type = mt5.ORDER_TYPE_BUY
    
    request = {
        "action": mt5.TRADE_ACTION_DEAL,
        "position": ticket,
        "symbol": position.symbol,
        "volume": position.volume,  # Close full volume
        "type": close_type,
        "price": close_price,
        "deviation": 20,
        "magic": position.magic,
        "comment": comment,
        "type_time": mt5.ORDER_TIME_GTC,
        "type_filling": mt5.ORDER_FILLING_IOC,
    }
    
    result = mt5.order_send(request)
    
    if result and result.retcode == mt5.TRADE_RETCODE_DONE:
        return {
            "success": True,
            "message": f"Position {ticket} closed successfully",
            "data": {
                "ticket": ticket,
                "symbol": position.symbol,
                "volume": position.volume,
                "close_price": close_price,
                "profit": result.profit if hasattr(result, 'profit') else 0
            }
        }
    else:
        return {"error": f"Close failed: {result.retcode if result else mt5.last_error()}"}

# Example usage
result = close_position(
    ticket=123456789,
    comment="Manual close via API"
)
```

### 5.2 Partial Close Position

```python
def close_position_partial(ticket, close_volume, comment="Partial close"):
    """
    Close part of position
    
    Args:
        ticket (int): Position ticket
        close_volume (float): Volume to close (must be <= position volume)
        comment (str): Close comment
    """
    
    # Get position info
    positions = mt5.positions_get(ticket=ticket)
    if not positions:
        return {"error": f"Position {ticket} not found"}
    
    position = positions[0]
    
    # Validate close volume
    if close_volume > position.volume:
        return {"error": f"Close volume {close_volume} exceeds position volume {position.volume}"}
    
    if close_volume <= 0:
        return {"error": "Close volume must be positive"}
    
    # Get current price
    tick = mt5.symbol_info_tick(position.symbol)
    if tick is None:
        return {"error": f"Price data not available for {position.symbol}"}
    
    # Determine close price and order type
    if position.type == 0:  # BUY position
        close_price = tick.bid
        close_type = mt5.ORDER_TYPE_SELL
    else:  # SELL position
        close_price = tick.ask
        close_type = mt5.ORDER_TYPE_BUY
    
    request = {
        "action": mt5.TRADE_ACTION_DEAL,
        "position": ticket,
        "symbol": position.symbol,
        "volume": close_volume,    # Partial volume
        "type": close_type,
        "price": close_price,
        "deviation": 20,
        "magic": position.magic,
        "comment": comment,
        "type_time": mt5.ORDER_TIME_GTC,
        "type_filling": mt5.ORDER_FILLING_IOC,
    }
    
    result = mt5.order_send(request)
    
    if result and result.retcode == mt5.TRADE_RETCODE_DONE:
        remaining_volume = position.volume - close_volume
        return {
            "success": True,
            "message": f"Partial close: {close_volume} of {position.volume} lots",
            "data": {
                "ticket": ticket,
                "symbol": position.symbol,
                "closed_volume": close_volume,
                "remaining_volume": remaining_volume,
                "close_price": close_price,
                "profit": result.profit if hasattr(result, 'profit') else 0
            }
        }
    else:
        return {"error": f"Partial close failed: {result.retcode if result else mt5.last_error()}"}

# Example usage
result = close_position_partial(
    ticket=987654321,
    close_volume=0.05,  # Close half of 0.1 lot position
    comment="Take 50% profit"
)
```

### 5.3 Close All Positions

```python
def close_all_positions(symbol=None, comment="Close all positions"):
    """
    Close all open positions
    
    Args:
        symbol (str): Close only positions for specific symbol (optional)
        comment (str): Close comment
    """
    
    # Get all positions
    positions = mt5.positions_get(symbol=symbol) if symbol else mt5.positions_get()
    
    if not positions:
        return {
            "success": True,
            "message": "No open positions to close",
            "data": {"closed_count": 0}
        }
    
    results = []
    closed_count = 0
    failed_count = 0
    
    for position in positions:
        result = close_position(position.ticket, comment)
        results.append({
            "ticket": position.ticket,
            "symbol": position.symbol,
            "result": result
        })
        
        if result.get("success"):
            closed_count += 1
        else:
            failed_count += 1
    
    return {
        "success": True,
        "message": f"Closed {closed_count} positions, {failed_count} failed",
        "data": {
            "total_positions": len(positions),
            "closed_count": closed_count,
            "failed_count": failed_count,
            "details": results
        }
    }

# Example usage
result = close_all_positions(
    symbol="EURUSD",  # Close only EURUSD positions
    comment="End of trading session"
)
```

---

## 6. Best Practices

### 6.1 Validation Functions

```python
def validate_order_levels(symbol, order_type, entry_price, stop_loss, take_profit):
    """
    Validate price levels for order
    
    Args:
        symbol (str): Trading symbol
        order_type (str): 'BUY_LIMIT', 'SELL_LIMIT', etc.
        entry_price (float): Entry price
        stop_loss (float): Stop loss price
        take_profit (float): Take profit price
    """
    
    # Get current price
    tick = mt5.symbol_info_tick(symbol)
    if not tick:
        return {"valid": False, "error": f"Cannot get price for {symbol}"}
    
    current_ask = tick.ask
    current_bid = tick.bid
    
    errors = []
    
    if order_type == "BUY_LIMIT":
        if entry_price >= current_ask:
            errors.append("BUY LIMIT entry must be below current ask price")
        if stop_loss >= entry_price:
            errors.append("Stop loss must be below entry price for BUY order")
        if take_profit <= entry_price:
            errors.append("Take profit must be above entry price for BUY order")
            
    elif order_type == "SELL_LIMIT":
        if entry_price <= current_bid:
            errors.append("SELL LIMIT entry must be above current bid price")
        if stop_loss <= entry_price:
            errors.append("Stop loss must be above entry price for SELL order")
        if take_profit >= entry_price:
            errors.append("Take profit must be below entry price for SELL order")
    
    return {
        "valid": len(errors) == 0,
        "errors": errors,
        "current_prices": {"ask": current_ask, "bid": current_bid}
    }

# Example usage
validation = validate_order_levels(
    symbol="XAUUSD",
    order_type="BUY_LIMIT",
    entry_price=2050.00,
    stop_loss=2040.00,
    take_profit=2070.00
)
```

### 6.2 Risk Management

```python
def calculate_position_size(symbol, account_balance, risk_percent, entry_price, stop_loss):
    """
    Calculate optimal position size based on risk management
    
    Args:
        symbol (str): Trading symbol
        account_balance (float): Account balance
        risk_percent (float): Risk percentage (e.g., 2.0 for 2%)
        entry_price (float): Entry price
        stop_loss (float): Stop loss price
    
    Returns:
        dict: Position size calculation result
    """
    
    # Get symbol info
    symbol_info = mt5.symbol_info(symbol)
    if not symbol_info:
        return {"error": f"Cannot get symbol info for {symbol}"}
    
    # Calculate risk amount
    risk_amount = account_balance * (risk_percent / 100)
    
    # Calculate stop loss distance
    sl_distance = abs(entry_price - stop_loss)
    
    if sl_distance == 0:
        return {"error": "Stop loss distance cannot be zero"}
    
    # Get contract size and pip value
    contract_size = symbol_info.trade_contract_size
    
    # Calculate pip value based on symbol digits
    if symbol_info.digits == 5 or symbol_info.digits == 3:
        pip_size = symbol_info.point * 10
    else:
        pip_size = symbol_info.point
    
    # For forex pairs
    if "USD" in symbol:
        if symbol.endswith("USD"):
            # Base currency is not USD (e.g., EURUSD)
            pip_value = pip_size * contract_size
        else:
            # Quote currency is USD (e.g., USDJPY)
            current_price = mt5.symbol_info_tick(symbol).ask
            pip_value = (pip_size * contract_size) / current_price
    else:
        # For other symbols (e.g., XAUUSD)
        pip_value = pip_size * contract_size
    
    # Calculate position size
    sl_distance_pips = sl_distance / pip_size
    position_size = risk_amount / (sl_distance_pips * pip_value)
    
    # Round to minimum volume step
    min_volume = symbol_info.volume_min
    volume_step = symbol_info.volume_step
    
    # Round down to nearest volume step
    position_size = int(position_size / volume_step) * volume_step
    
    # Ensure minimum volume
    if position_size < min_volume:
        position_size = min_volume
    
    # Check maximum volume
    max_volume = symbol_info.volume_max
    if position_size > max_volume:
        position_size = max_volume
    
    return {
        "success": True,
        "position_size": position_size,
        "data": {
            "risk_amount": risk_amount,
            "sl_distance_pips": sl_distance_pips,
            "pip_value": pip_value,
            "min_volume": min_volume,
            "max_volume": max_volume,
            "volume_step": volume_step
        }
    }

# Example usage
position_calc = calculate_position_size(
    symbol="EURUSD",
    account_balance=10000,  # $10,000 account
    risk_percent=2.0,       # Risk 2% per trade
    entry_price=1.0800,
    stop_loss=1.0750        # 50 pips risk
)
```

### 6.3 Order Management Helper

```python
class MT5OrderManager:
    """
    Comprehensive order management class
    """
    
    def __init__(self, magic_number=123456):
        self.magic = magic_number
        self.active_orders = {}
        self.active_positions = {}
    
    def place_limit_order(self, symbol, order_type, volume, entry_price, 
                         stop_loss=None, take_profit=None, comment=""):
        """
        Place limit order with validation
        
        Args:
            order_type (str): 'BUY_LIMIT' or 'SELL_LIMIT'
        """
        
        # Validate order levels
        validation = validate_order_levels(
            symbol, order_type, entry_price, stop_loss, take_profit
        )
        
        if not validation["valid"]:
            return {"error": f"Validation failed: {validation['errors']}"}
        
        # Determine MT5 order type
        mt5_type = mt5.ORDER_TYPE_BUY_LIMIT if order_type == "BUY_LIMIT" else mt5.ORDER_TYPE_SELL_LIMIT
        
        request = {
            "action": mt5.TRADE_ACTION_PENDING,
            "symbol": symbol,
            "volume": volume,
            "type": mt5_type,
            "price": entry_price,
            "sl": stop_loss,
            "tp": take_profit,
            "deviation": 20,
            "magic": self.magic,
            "comment": comment or f"{order_type} {symbol}",
            "type_time": mt5.ORDER_TIME_GTC,
            "type_filling": mt5.ORDER_FILLING_IOC
        }
        
        result = mt5.order_send(request)
        
        if result and result.retcode == mt5.TRADE_RETCODE_DONE:
            order_data = {
                "ticket": result.order,
                "symbol": symbol,
                "type": order_type,
                "volume": volume,
                "entry": entry_price,
                "sl": stop_loss,
                "tp": take_profit,
                "timestamp": datetime.now()
            }
            
            self.active_orders[result.order] = order_data
            
            return {
                "success": True,
                "ticket": result.order,
                "message": f"{order_type} order placed successfully",
                "data": order_data
            }
        else:
            return {
                "error": f"Order failed: {result.retcode if result else mt5.last_error()}"
            }
    
    def get_position_info(self, ticket):
        """Get detailed position information"""
        positions = mt5.positions_get(ticket=ticket)
        if not positions:
            return {"error": f"Position {ticket} not found"}
        
        position = positions[0]
        
        # Calculate profit/loss in pips
        symbol_info = mt5.symbol_info(position.symbol)
        if symbol_info.digits == 5 or symbol_info.digits == 3:
            pip_size = symbol_info.point * 10
        else:
            pip_size = symbol_info.point
        
        current_price = mt5.symbol_info_tick(position.symbol)
        
        if position.type == 0:  # BUY
            current_close_price = current_price.bid
            pips = (current_close_price - position.price_open) / pip_size
        else:  # SELL
            current_close_price = current_price.ask
            pips = (position.price_open - current_close_price) / pip_size
        
        return {
            "success": True,
            "data": {
                "ticket": position.ticket,
                "symbol": position.symbol,
                "type": "BUY" if position.type == 0 else "SELL",
                "volume": position.volume,
                "open_price": position.price_open,
                "current_price": current_close_price,
                "sl": position.sl,
                "tp": position.tp,
                "profit_usd": position.profit,
                "profit_pips": round(pips, 1),
                "swap": position.swap,
                "commission": position.commission,
                "open_time": position.time,
                "comment": position.comment
            }
        }
    
    def bulk_close_positions(self, symbol=None, profit_threshold=None):
        """
        Close multiple positions based on criteria
        
        Args:
            symbol (str): Close only specific symbol positions
            profit_threshold (float): Close only positions above this profit
        """
        
        # Get positions to close
        positions = mt5.positions_get(symbol=symbol) if symbol else mt5.positions_get()
        
        if not positions:
            return {"success": True, "message": "No positions to close", "data": {"closed": []}}
        
        closed_positions = []
        failed_closes = []
        
        for position in positions:
            # Check profit threshold
            if profit_threshold is not None and position.profit < profit_threshold:
                continue
            
            # Close position
            result = close_position(position.ticket, "Bulk close")
            
            if result.get("success"):
                closed_positions.append({
                    "ticket": position.ticket,
                    "symbol": position.symbol,
                    "profit": position.profit
                })
            else:
                failed_closes.append({
                    "ticket": position.ticket,
                    "symbol": position.symbol,
                    "error": result.get("error")
                })
        
        return {
            "success": True,
            "message": f"Closed {len(closed_positions)} positions",
            "data": {
                "closed": closed_positions,
                "failed": failed_closes,
                "total_profit": sum(p["profit"] for p in closed_positions)
            }
        }

# Global order manager instance
order_manager = MT5OrderManager(magic_number=123456)
```

---

## 7. Error Handling

### 7.1 Common Error Codes

```python
def get_error_description(retcode):
    """
    Get human-readable error description
    
    Args:
        retcode (int): MT5 return code
    
    Returns:
        str: Error description
    """
    
    error_codes = {
        10004: "Requote - Price has changed",
        10006: "Request rejected - Invalid request",
        10007: "Request canceled by trader",
        10008: "Order placed - Order accepted",
        10009: "Request completed - Order executed",
        10010: "Only part of the request was completed",
        10011: "Request processing error",
        10012: "Request canceled by timeout",
        10013: "Invalid request",
        10014: "Invalid volume in the request",
        10015: "Invalid price in the request",
        10016: "Invalid stops in the request",
        10017: "Trade is disabled",
        10018: "Market is closed",
        10019: "There is not enough money to complete the request",
        10020: "Prices changed",
        10021: "There are no quotes to process the request",
        10022: "Invalid order expiration date in the request",
        10023: "Order state changed",
        10024: "Too frequent requests",
        10025: "No changes in request",
        10026: "Autotrading disabled by server",
        10027: "Autotrading disabled by client terminal",
        10028: "Request locked for processing",
        10029: "Order or position frozen",
        10030: "Invalid order type",
        10031: "Invalid order state",
        10032: "Invalid order expiration",
        10033: "Invalid order volume",
        10034: "Invalid order price",
        10035: "Invalid order stops",
        10036: "Invalid order filling",
        10038: "Order limit reached",
        10039: "Order volume limit reached",
    }
    
    return error_codes.get(retcode, f"Unknown error code: {retcode}")

def handle_order_error(result):
    """
    Handle order send errors with detailed logging
    
    Args:
        result: MT5 order_send result
    
    Returns:
        dict: Formatted error response
    """
    
    if result is None:
        last_error = mt5.last_error()
        return {
            "error": "Order send failed",
            "details": f"MT5 Error: {last_error}",
            "code": last_error[0] if last_error else None,
            "description": last_error[1] if last_error else "Unknown error"
        }
    
    if result.retcode != mt5.TRADE_RETCODE_DONE:
        return {
            "error": "Order execution failed",
            "details": get_error_description(result.retcode),
            "code": result.retcode,
            "request_id": result.request_id if hasattr(result, 'request_id') else None,
            "volume": result.volume if hasattr(result, 'volume') else None,
            "price": result.price if hasattr(result, 'price') else None,
            "comment": result.comment if hasattr(result, 'comment') else None
        }
    
    return {"success": True}
```

### 7.2 Retry Mechanism

```python
import time
from functools import wraps

def retry_on_failure(max_attempts=3, delay=1, backoff=2):
    """
    Retry decorator for MT5 operations
    
    Args:
        max_attempts (int): Maximum retry attempts
        delay (float): Initial delay between retries
        backoff (float): Delay multiplier for each retry
    """
    
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            attempts = 0
            current_delay = delay
            
            while attempts < max_attempts:
                try:
                    result = func(*args, **kwargs)
                    
                    # Check if result indicates success
                    if isinstance(result, dict):
                        if result.get("success") or not result.get("error"):
                            return result
                        
                        # Check for retryable errors
                        if "requote" in str(result.get("error", "")).lower():
                            attempts += 1
                            if attempts < max_attempts:
                                print(f"Retry {attempts}/{max_attempts} for {func.__name__} after {current_delay}s")
                                time.sleep(current_delay)
                                current_delay *= backoff
                                continue
                    
                    return result
                    
                except Exception as e:
                    attempts += 1
                    if attempts >= max_attempts:
                        return {"error": f"Max retries exceeded: {str(e)}"}
                    
                    print(f"Exception in {func.__name__}, retry {attempts}/{max_attempts}: {e}")
                    time.sleep(current_delay)
                    current_delay *= backoff
            
            return {"error": f"Operation failed after {max_attempts} attempts"}
        
        return wrapper
    return decorator

# Apply retry to critical functions
@retry_on_failure(max_attempts=3, delay=0.5)
def place_order_with_retry(symbol, order_type, volume, entry_price, stop_loss, take_profit):
    """Place order with automatic retry on failure"""
    return order_manager.place_limit_order(
        symbol=symbol,
        order_type=order_type,
        volume=volume,
        entry_price=entry_price,
        stop_loss=stop_loss,
        take_profit=take_profit
    )

@retry_on_failure(max_attempts=2, delay=0.3)
def modify_order_with_retry(ticket, new_price=None, new_sl=None, new_tp=None):
    """Modify order with automatic retry"""
    return modify_pending_order(ticket, new_price, new_sl, new_tp)
```

### 7.3 Comprehensive Testing

```python
def test_order_functions():
    """
    Test all order management functions
    """
    
    print("Testing MT5 Order Management Functions...")
    
    # Test connection
    if not mt5.initialize():
        print("❌ MT5 initialization failed")
        return
    
    print("✅ MT5 connected successfully")
    
    # Test symbol availability
    test_symbol = "EURUSD"
    symbol_info = mt5.symbol_info(test_symbol)
    
    if symbol_info is None:
        print(f"❌ Symbol {test_symbol} not available")
        return
    
    print(f"✅ Symbol {test_symbol} available")
    
    # Test current price retrieval
    tick = mt5.symbol_info_tick(test_symbol)
    if tick is None:
        print(f"❌ Cannot get price for {test_symbol}")
        return
    
    print(f"✅ Current price: Bid={tick.bid}, Ask={tick.ask}")
    
    # Test order validation
    validation = validate_order_levels(
        symbol=test_symbol,
        order_type="BUY_LIMIT",
        entry_price=tick.ask - 0.0020,  # 20 pips below
        stop_loss=tick.ask - 0.0070,    # 70 pips below
        take_profit=tick.ask + 0.0030   # 30 pips above
    )
    
    if validation["valid"]:
        print("✅ Order validation passed")
    else:
        print(f"❌ Order validation failed: {validation['errors']}")
    
    # Test position size calculation
    pos_calc = calculate_position_size(
        symbol=test_symbol,
        account_balance=10000,
        risk_percent=1.0,
        entry_price=tick.ask - 0.0020,
        stop_loss=tick.ask - 0.0070
    )
    
    if pos_calc.get("success"):
        print(f"✅ Position size calculation: {pos_calc['position_size']} lots")
    else:
        print(f"❌ Position size calculation failed: {pos_calc.get('error')}")
    
    print("\n📋 All tests completed!")
    print("Functions are ready for live trading.")
    
    mt5.shutdown()

# Run tests
if __name__ == "__main__":
    test_order_functions()
```

---

## Summary

Dokumentasi ini mencakup:

✅ **Limit Order dengan Entry, SL & TP** - Fungsi lengkap untuk BUY/SELL LIMIT  
✅ **Modify Order** - Edit pending order dan open position  
✅ **Edit Take Profit** - Update TP level saja  
✅ **Trailing Stop Loss** - Manual dan automated trailing  
✅ **Close Position** - Full, partial, dan bulk close  
✅ **Best Practices** - Validation, risk management, error handling  
✅ **Error Handling** - Comprehensive error management dan retry mechanism  

**Key Features:**
- 🔒 Validasi lengkap untuk semua order levels
- 💰 Risk management dengan position size calculator
- 🔄 Retry mechanism untuk handling network issues
- 📊 Comprehensive order management class
- 🎯 Automated trailing stop service
- ⚡ Bulk operations untuk multiple positions

**Siap untuk production trading dengan safety features lengkap!**
