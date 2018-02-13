    operation = (name) -> ({op}) -> op is name
    Value = ({value}) -> value
    Key = ({key}) -> key
    is_string = (v) -> typeof v is 'string'
    is_object = (v) -> typeof v is 'object'
    not_null = (v) -> v?

    module.exports = {operation,Value,Key,is_string,is_object,not_null}
