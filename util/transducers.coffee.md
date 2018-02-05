    operation = (name) -> ({operation}) -> operation is name
    Value = ({value}) -> value
    Key = ({key}) -> key
    is_string = (v) -> typeof v is 'string'
    is_object = (v) -> typeof v is 'object'
    not_null = (v) -> v?

